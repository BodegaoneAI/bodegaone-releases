# Changelog

All notable changes to Bodega One are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0-beta.41] - 2026-09-03

Sub-agents. The agent can hand a focused task to another agent that runs on its own provider and
model and get its report back: a built-in `delegate` that edits inside a throwaway copy of your
project with its diff verified before anything lands, a read-only `researcher`, and any custom agent
you mark as spawnable. A sub-agent can run in the background while you keep working, and you can
click it to watch every tool call as it happens. Off by default, with its own spend cap, and only one
local sub-agent at a time because two models generating on one GPU is where things fall over. Before
it shipped it was driven hard across local and cloud model combinations on a single 5090: eighteen
defects found that way are fixed here, and the whole branch passed the full local gate.

### Added

- **Sub-agents.** The agent can hand a focused task to another agent that runs on its own provider and
  model, and get its report back. Two built-ins: `delegate` edits inside a throwaway copy of your project
  and its diff is verified before anything lands, `researcher` runs read-only and reports. Any custom
  agent can be one too: Settings → Custom agents now has Runs on, When spawned as a sub-agent, Reasoning
  effort, and a switch for whether the main agent may spawn it. Off by default under Settings → Agent,
  with its own per-sub-agent spend cap, because several local models on one GPU is where things fall over.
  A sub-agent cannot spawn sub-agents, cannot use the shell unless you opt in, and a cloud one is refused
  under air-gap. This replaces `delegate_subtask`, whose settings carry over as the `delegate` built-in.
- **A sub-agent can run in the background.** The agent hands off a task and keeps working; the report
  arrives on a later turn rather than blocking. A strip under the conversation shows each sub-agent
  while it runs, what it did when it finished, and what it cost — a sub-agent whose work failed
  verification says its changes were discarded rather than reading like a success. Only one sub-agent
  on a local provider runs at a time, whatever the overall limit is, because two models generating on
  one GPU is the thing that falls over. A background sub-agent may outlive the turn that started it
  and report on the next one; stopping the run, or the run failing, stops its sub-agents.
- **A run's spend cap covers its sub-agents, and it counts as you go.** The cap is what the whole run may
  cost, sub-agents included, and a run stopped by it says how much of the total went to them. Cloud
  spend used to be recorded only after a run finished, so a cap could only see money already spent
  by something else; the run's own calls are now estimated live, which is what lets a sub-agent's own
  smaller cap actually stop it mid-run.
- **Unattended runs can use sub-agents, if you ask.** They are off by default in the CLI, Loops and
  background runs, because nobody is watching one and a run that can spawn its own agents compounds
  time and spend. `bodega run --subagents` turns them on for a single run; there is a matching switch
  under Settings. A sub-agent still can never spawn a sub-agent — that limit is enforced separately and
  no setting relaxes it.
- **Exporting a custom agent now exports how it was configured.** The JSON bundle was dropping the
  agent's model provider and its sub-agent settings, so an agent came back from its own export blanked.
  It carries them now. It still never carries whether the main agent may spawn it: an imported agent
  arrives inert and you turn that on yourself.
- **A cloud model can delegate to a local one.** `delegate_subtask` used to refuse whenever your main
  provider was a cloud one, because it checked *your* provider rather than the delegate's. A new
  setting under Mixture, "Delegate runs on", picks the sub-agent's provider (Ollama first: it serves
  any pulled model by name without unloading yours), with the model named next to it. The sub-agent
  runs on its own provider inside its worktree, and its usage is now recorded against your session,
  so a run's spend cap counts it. First slice of the sub-agent work.

- **Click a sub-agent to watch it work.** The strip under the conversation now opens: each
  sub-agent shows which agent and model it is, how long it has run, what it has cost, every tool
  call as it happens, and its report when it finishes. What a tool returned never leaves the
  backend; only the call itself is shown.
- **The agent knows which sub-agents it may use.** Its prompt now lists the two built-ins and every
  custom agent you have marked as spawnable, with the provider and model each one runs on. Before
  this, a custom agent's name was never told to the model, so asking it to use one by name often did
  nothing.
- **A sub-agent that stopped for a reason says so.** When a sub-agent hits its own spend or iteration
  cap, the main agent is told why it stopped instead of only that it produced nothing.
- **A sub-agent lost to a restart is reported, not forgotten.** If Bodega restarts while a
  background sub-agent is running, your next reply says that it was lost and that none of its work
  was applied.
- **`bodega agents`** in the CLI lists your custom agents, where each one runs, and whether the main
  agent may spawn it.

### Fixed

- **A custom agent picked as the main agent now runs on its own provider.** A DeepSeek or OpenRouter
  agent chosen from the agent dropdown was being sent to whichever provider was active, and failed
  with "model not found". The agent's own provider wins now.
- **A sub-agent's fix is no longer thrown away because your test suite was already red.** The check
  that verifies a sub-agent's change runs your project's tests, and it used to refuse on any
  failure, including ones that were failing before the sub-agent touched anything. It now records
  which tests were already failing and only refuses when the sub-agent's change breaks something
  new.
- **OpenRouter costs were sometimes wildly overstated.** A run on OpenRouter could be billed at a
  placeholder rate instead of the model's real price, in one case a hundred times too much, which
  could also trip a spend cap for no reason. Live prices are now fetched before a cost is recorded.
- **Sub-agents now hold up under real, hands-on testing.** Live testing across local model
  combinations turned up fourteen problems and all fourteen are fixed: a sub-agent could go
  unnoticed by a smaller local model because it wasn't listed among the model's available tools;
  a local sub-agent could fail to load or could send the wrong model name to Ollama or llama.cpp;
  a sub-agent's correctly-fixed and tested code could get thrown away instead of applied, and its
  temporary working copy could get left behind on disk; calling a sub-agent now correctly asks for
  your approval first, the same as any other action that changes files; a background sub-agent's
  report now always arrives on your next reply, even if that reply is just answering a question;
  stopping a run now reliably cancels any sub-agent that was still working, instead of letting a
  cancelled sub-agent's changes get applied anyway; and a research-only sub-agent can now actually
  read your project's files, rather than having nothing useful available to it.
- **A sub-agent that doesn't fit your GPU's memory alongside your main model is now refused up
  front if it would run in the background** (rather than causing your GPU to keep swapping models
  back and forth on every message), while a foreground sub-agent is still allowed with a one-time
  model swap.

## [1.0.0-beta.40] - 2026-09-02

Runs are governed by progress now. The step and time limits became ceilings a run grows toward while
it keeps landing files, fixing tests and getting a build to compile, and a run that stops making
progress ends early and says why. Measured on three real projects against a local 27B model — a Rust
TUI that was deadline-cut with four compile errors before now finishes clean in 37 minutes after four
earned extensions. Around it: Fable 5.1 in the picker, a last-minute hardening audit (an API endpoint
that ran tools with no key, a git-hook edit that needed no approval), and twenty-two harness fixes
found by driving the shipped build and building its output by hand.

### Added

- **Fable 5.1 is in the model picker.** Same tier and price as Fable 5 ($10/$50 per million
  tokens), 1M context, and a 1M picker variant for parity with the rest of the family. Two things
  about this model are new and both are handled: cache reads cost a quarter of the rate every
  other model in the family uses, so cost estimates carry the real $0.25 figure rather than a
  derived one; and its reasoning is tied to the exact conversation that produced it, which
  Bodega's per-turn context refresh would otherwise trip on every second call. Bodega now asks the
  API to drop stale reasoning and continue instead of failing, and logs when that happens.

### Changed

- **Runs are now governed by progress, not by a fixed step count or clock.** The iteration limit and
  the time limit still exist as absolute ceilings, but a run that keeps making progress — new files,
  fewer failing tests, a build that starts compiling — earns more budget automatically, and a run that
  stops making progress ends early and says why. Cloud runs also get a per-run spend cap. Three new
  settings under Agent: the iteration ceiling, the time ceiling, and the spend cap. The model is told its
  remaining budget and recent progress every turn, so it can plan to finish rather than be cut off.
- **Settings → Agent shows the three run ceilings** (iterations, time, per-run spend), with ranges,
  in place of the Max Workers control that no longer governed anything. The help pages for Code
  Mode, AI panels, automations, CLI and spending now describe how a run's budget grows and stops.

### Fixed

- **The local API server's agentic endpoint now refuses to run without an API key.** With no key
  set, the built-in server accepted every request as "open access", and that endpoint runs tools on
  your machine. It now answers 403 until a key is set under Settings, and says so.
- **`str_replace` can no longer edit git hooks or `.git/config`.** The full-file write tool already
  refused those paths; a surgical edit to `.git/hooks/pre-commit` needed no approval in Act mode and
  would have run on your next commit.
- **`run_tests` and `delegate_subtask` count as writes.** In Ask mode they were treated as reads and
  ran the project's test script without asking.
- **MCP tools with nullable or union parameters work again.** A parameter typed `["string","null"]`
  or with `anyOf` was rejected before the tool ever ran.
- **The Embeddings rebuild button reports failure.** It logged the error and left the button as if
  nothing had happened.
- **The shell tool is described as running cmd.exe on Windows, because it does.** The prompt and
  the tool description said PowerShell, so models wrote PowerShell that failed.
- **Long non-streaming local calls (compaction summaries, memory extraction, the reflector) no
  longer fail as "Could not connect" after 8 seconds.** A non-streaming request sends no headers
  until generation finishes, and the connect timer wrapped the whole call.
- **The compactor's own "is the window full" check now uses the same rule the callers use.** It
  measured message content only, so a tool-heavy history read as not full and compaction did nothing
  until the hard trim did the work.
- **The per-turn budget line no longer breaks the prompt cache or the reasoning round.** It rode as a
  trailing system message; Anthropic folds every system message into one string that then changed
  every turn, and the OpenAI-compatible wire demoted it to a user turn. It now lives in the per-turn
  block that already changes every turn.
- **Reattaching to a run in progress keeps the finish frame's details.** After a reconnect, an error
  finish or a forced stop read as a clean finish because only the text made it through.
- **A `-pro` or `-fast` variant OpenAI does not have a price for no longer inherits the base
  model's rate.** `o3-pro` was billed at the `o3` rate. The reverse match in the boost tracker, which
  priced a short id at whatever longer key happened to start with it, is gone too.
- **The fleet "files changed" count only counts writes.** Reads and directory listings were counted
  as changed files.
- **The "extend the run?" card no longer feeds the approval-learning ledger or the rejection circuit
  breaker.** Declining it was recorded as a human "no" to a tool.
- **The null-turn recovery leaves room for the prompt when it raises `max_tokens`.** The ceiling was
  half the window with no prompt term, so the raise could be silently truncated.
- **A run the repetition guard cut short now says so.** When the agent kept making the same edit or
  the same call, the loop stopped it and graded what existed, but the answer read like a finish the
  model chose. It now carries a trailer naming the repetition, runs the build check the other forced
  stops run, and records the reason on the message so the CLI reports it too.
- **Reconnecting to a run that died mid-stream no longer looks like a clean finish.** If the
  connection closed before the run's final frame arrived, the partial text is now marked as possibly
  incomplete instead of being shown as the answer.

- **The "model loaded on the CPU" notice now reaches the screen.** The previous release detected a
  quiet CPU fallback and wrote it to the log, and its notes said you would see it. You would not have:
  nothing in the app listened for it. It now appears in the chat status line, with the likely cause
  and what to do, whether the model loaded during your first message or earlier from the Models page.
- **Rust projects get an automatic build check after the agent writes code.** TypeScript, Python and
  Go projects already did; a Cargo project got nothing, so a broken build could reach "done" unless
  the model thought to run cargo itself. Bodega now runs `cargo check` after writes and hands the
  compiler's errors back to the model to fix.
- **Act mode stops cutting a capable model's run short on its third edit to one file, and stops turning
  its edits into reads.** Two more rails from the same family as last release's fix still decided by
  whether a window was attached rather than by the mode you chose. Both now follow the mode. A small
  or weak model keeps every rail, as before.
- **The agent is no longer told to stop working while it is fixing a failing build.** After three
  read-only turns following a write, Bodega used to tell the model that everything had been created
  successfully and to write its final answer — including when the model was reading files to fix
  compile errors it had just been handed. That message now waits while a build is failing, and no
  longer claims a success nothing checked.
- **A local model's context is no longer two-thirds old thinking.** On llama.cpp and other
  OpenAI-compatible hosts, Bodega replayed the model's reasoning from every previous turn back into
  the prompt. Qwen's own guidance is not to, and on a 57k-token window that was the difference between
  compacting at iteration 18 and not compacting at all. Reasoning is now sent back only for the current
  round, which is what the one provider that requires it (DeepSeek) actually needs.
- **The context estimate is calibrated to what the model actually counts.** Bodega guessed 2.6
  characters per token for everything; this run's Rust code measured 3.82 on Qwen3, so the estimate ran
  1.5x high, reported "over 90% full" at two-thirds, and on that false reading trimmed history — and
  dropped the original task — thirteen turns in a row. llama.cpp is now asked to report real prompt
  sizes (it always could), the estimate calibrates to them after the first call, and the overflow path
  never drops the message that holds the task. The same over-count lived in the mid-run compaction
  budget, which read a one-fifth-full window as 127–196% full and reported "nothing to compact" three
  times: earlier rounds' thinking is not resent to the model, and is no longer counted as if it were.
- **The end-of-run file list names each file once.** It used to list every write, so a file rewritten
  three times appeared three times and the count was of writes, not files.
- **"Scaffold the whole project" now means something concrete.** A from-scratch prompt that named no
  files produced a contract with nothing to check, so a project with a 54-byte main file could be
  reported as finished. Such prompts now require the ecosystem's manifest and entry point (and the
  README when asked for), and a placeholder entry point does not count as written.
- **Installing or adding a dependency from the shell now re-grounds the model on the project's pinned
  versions.** The version note used to fire only after Bodega wrote a manifest itself; `cargo add`,
  `npm install` or `pip install` from the shell left the model guessing at APIs for versions it
  had just installed.
- **A turn spent entirely on thinking no longer nudges the sampler as if the run were stuck.** The
  recovery for that case already changes the next call; heating the temperature on top of it made
  the retry different for the wrong reason.
- **"Tell me the results" no longer turns a build into an edit.** A prompt that asked Bodega to
  scaffold a project and finished with "run the tests and tell me the results" was classified as a
  modification of existing files, which switched off the per-file proof checks that creation tasks
  get. The opening request now decides what kind of task it is.
- **The build check at the end of a run reads coloured test output correctly.** pytest prints its
  summary in colour even when asked not to, and the colour codes hid the failure count from the
  check, so a run that ended with 4 failing tests was reported as "Build check: passed". Colour is
  now stripped before the result is read, and pytest is told not to colour at all.
- **An approval nobody answered is no longer counted as a "no".** When a tool approval sat
  unanswered for five minutes, Bodega recorded it exactly like a rejection: it told the model
  "rejected by user" and counted it toward the rule that warns about a tool after three
  rejections. A timer running out is not a decision. The model is now told nobody answered, and
  nothing is learned from it.
- **The Code Mode permission mode survives a reload.** Ask, Plan or Act was saved but never read
  back, so every launch quietly reset the panel to Ask.
- **Pressing Stop no longer shows "Request timed out".** Stopping a run was reported as the
  model taking too long, with advice to pick a faster model. A stop now just ends the turn.
- **A tool call cut off by the output budget no longer kills the session.** When a local model ran
  out of output tokens partway through writing a file, the half-written call was replayed to the
  model on every later turn, and llama.cpp refused each request outright. A seven-minute run
  ended with three files and a raw server error on screen. The cut-off call is now replaced with a
  short note of what happened before it goes back to the model, so the run continues.
- **A run cut by the time limit says so, and reports the build.** The time-limit exit recorded itself
  but showed a mid-sentence answer with no note; only the iteration limit had one. Both now do, and on
  a forced stop in a code project Bodega runs the project's build check and appends the result, so a
  cut-off run ends with "build: 4 errors" rather than silence.
- **The agent is told which library versions the project actually pins.** On the first write in a
  project with a manifest, Bodega reads the lockfile (or the manifest) and hands the model the exact
  versions of its direct dependencies, with where to look if it is unsure an API exists in that
  version. The note repeats only when the pins change. This was the cause of the last measured run's
  twenty compile errors: the model had pinned one version of a library and written code for another
  from memory, and nothing told it.
- **Gateway providers now bill at the gateway's own price, not the upstream model's.** A call through
  DeepInfra, Fireworks, Together, Novita or SambaNova used to be priced at whatever the model's
  original vendor charges, because the price table only knew model names. DeepInfra charges 50%
  more than Anthropic for Sonnet 5, Fireworks charges a fraction of Moonshot's rate for Kimi, and
  both were billed wrong in opposite directions. Each of those providers now has its own price
  list, sourced from its own pricing page, and a model it doesn't list shows as unpriced rather
  than borrowing a number. DeepInfra, Novita, SambaNova and Together also get a short curated model
  list of coding models with a published rate.
- **GPT-5.6 Sol was billing at its old rate.** OpenAI cut Sol to $4.00 per million input tokens
  and $20.00 per million output, and Bodega was still charging the previous $5.00/$30.00 — 25%
  over on input and 50% over on output. Cost estimates and the price badge both move; they had
  agreed with each other and were wrong together. Found by a new weekly check against the
  providers' own pricing pages, which is the only thing that can catch this class: the rate was
  correct the day it was written, and went stale because the vendor changed it, so no test in the
  codebase could have failed.

---

## [1.0.0-beta.39.1] - 2026-09-01

A pure-fix release. No new features — ten harness fixes found by driving the shipped beta.39
against a local 27B model end to end, each verified against the beta.39 tag rather than against
development. The headline is that an agent in Act mode could report an hour of tool calls and
leave nothing on disk.

### Fixed

- **Act mode actually grants write autonomy in the app.** Selecting the mode the app labels
  "Act — Full autonomy to complete tasks" left every chaperone rail in place, because the rail
  that stands down required the run to be *headless* — and a desktop session never is. The
  practical result was runs that reported tool calls for hours and wrote **zero files**: a
  contract gate answered every file write whose name the prompt had not happened to mention with
  a silent "skipped". The write whitelist now keys on the autonomy you granted rather than on
  whether a window is attached. The capability gate, `agent.force_guardrails`, and the stricter
  rule for spawning prompt-derived shell commands are unchanged.

- **A turn that runs out of budget mid-thought is retried differently, not identically.** On a
  model whose reasoning is longer than its output budget, every turn was truncated by
  construction and returned nothing — and the retry re-sent the same request, so it failed the
  same way. The loop now raises the output budget to the context window's ceiling first (keeping
  the reasoning intact), suppresses reasoning only when a larger budget was already tried or
  there is no headroom to try one, and strips dead reasoning from history last.

- **A tool call cut off mid-JSON is recoverable, not a dead run.** Generation stopping inside a
  tool call's arguments produced a parse error that nothing in the loop recognised, killing the
  run outright. It is now detected — on both the arguments framing *and* the parse failure, so
  genuine server faults stay terminal — and retried once with reasoning actually disabled on the
  wire.

- **A starved context exits honestly instead of grinding.** Three consecutive material
  post-compaction trims now end the run with a stated reason and a user-visible "Context
  pressure" line, rather than looping on a window that can no longer hold the task.

- **"Thinking: Off" now actually turns thinking off.** On llama.cpp and Qwen models, the reasoning
  effort chip was checked before the off switch, so whenever a tier was selected — and the chip
  offers no "off" — the request went out with thinking enabled regardless. Choosing off is a
  decision, not a preference, and it now wins. Leaving the dial untouched still lets the model's
  own default stand.

- **A model that is working is no longer cut off for "talking too much."** The cap that stops a
  model narrating instead of acting was also counting the nudges sent to a model that *was*
  acting — so a run that had written its files and was fixing its own build errors could be
  stopped mid-repair, a third of the way through its budget. The cap now counts only prose that
  arrived instead of work.

- **A model that quietly loaded on the CPU now says so.** When another program is holding video
  memory, llama.cpp can fall back to the CPU, answer normally, and report itself ready — while
  running many times slower, with nothing on screen explaining why. That fallback is now detected
  and named in the log, with the likely cause (another app holding video memory) and what to do
  about it. A deliberate CPU run, and the normal partial split for a model slightly larger than the
  card, stay silent. *(Corrected 2026-09-01: in this release the notice reached the log only, not
  the screen — see the next release's entry.)*

## [1.0.0-beta.39] - 2026-08-31

### Fixed

- **Your messages keep their `<` and `>` characters.** A server-side safety filter was silently
  deleting every angle bracket from everything you sent — pasted code lost its generics
  (`Array<string>` became `Array string `), JSX was mangled, and file attachments broke because
  the attachment wrapper itself was stripped, which is what made attaching a document trip the
  "message too long" error. Message content now passes through untouched; the filter still
  applies to non-content fields like titles and names.

- **Attaching a document no longer trips the message-length limit.** Attaching a file through
  the composer could get the whole message rejected with "exceeds maximum length" — the input
  guard was counting the document's content as if you had typed it. Attached files are exempt
  (the model's context window is the real limit); an over-long typed message is still refused.
- **The "Model not loaded" notice can be dismissed.** The banner that appears when a model's
  previous load never finished had no close button — the only way out was loading the model.
  It now has one, and dismissing it only hides the notice for that model; a different blocked
  model still shows it.

- **Bodega Routing now routes the way the Routing page describes it.** Four things it promised but
  didn't do: a step that ran a shell command (a `git commit`, an `npm install`) was treated as a
  "read" and handed to your fast model — it now stays on your smart model, like any other step we
  can't call safe. The list of "read" tools was also missing most of the real ones, so code search,
  symbol lookup, diagnostics, diffs and doc lookups never reached your fast model; they do now, and
  two names in that list had been dead for so long they could never match anything. Steps where
  Bodega is re-checking failed work now genuinely count as verification, which means a rule you
  wrote for `verify` steps will finally fire instead of sitting there doing nothing. And a routing
  rule limited to local-only models now reads the model your session is actually on, rather than
  whatever provider is set globally.

### Changed

- **Settings → Routing describes routing honestly.** The page claimed every routed step is shown
  and never a silent switch. What actually happens: the step list appears under the reply once a
  run finishes, only when the run used more than one model, and only for that run — it isn't kept
  once you send again. The page now says that, and says plainly that shell steps stay on your
  smart model.

- **The licensing copy now says what is actually true: the beta is free.** Settings → About and
  the README used to mention a one-time commercial price. That language is gone while we decide
  how to license the source. Bodega One is free for everyone during the public beta — the only
  ask is that you don't redistribute, resell or rebrand it.

### Added

- **Device-Frame Preview picks the right content model for ordinary websites.** A new default
  "Browser" mode renders the page below the drawn status bar and above the home indicator —
  the way a site looks in Safari — with the viewport size reported honestly and no phantom
  safe-area insets. "Edge to edge" keeps the full-bleed model for apps that handle insets
  themselves. Desktop scrollbars are hidden while a device frame is active.

### Fixed

- **Clicking a search result in Settings now takes you to the setting.** Searching Settings
  for, say, "safety" and clicking **Privacy & Safety** left the search results covering the
  page — you had to clear the search box with the X before the section appeared. Picking a
  section from a search now ends the search and shows the page. Help &amp; Docs works the same
  way, and a docs result that matched a specific topic now scrolls to that topic instead of
  the top of the page.
- **A tool approval could appear with no way to answer it.** Asking the agent for something
  that needed your permission could leave it sitting on "Thinking" until you restarted the
  session — the approval was waiting, but no Allow or Reject ever appeared. Approvals now
  always offer a decision, are never hidden behind an expander, and say plainly when the
  tool's details didn't come through.
- **The Agent, Research, Debug and Advisor panel tabs are announced correctly** by screen
  readers.
- **Opening the Agent Browser no longer crushes the Agent panel.** The panel could be
  squeezed to a sliver, pushing the Debug and Advisor tabs off the edge of the window with
  nothing on screen to suggest the tab strip scrolled — two tabs unreachable, and no way to
  guess they were there. The panel now keeps a minimum width, and a tab strip that scrolls
  shows that it does.
- **Side-loaded GGUF models are now inspected on import.** Bodega reads the model's own chat
  template to tell whether it supports tool calling, instead of assuming it doesn't — models
  that need their own template now get it. Where Bodega can't tell, the model is marked
  "plain template" in My Models with the reason, rather than quietly producing worse results.
  Models you already added are re-checked automatically.
- **A session with no project attached no longer reports a git branch.** The status line
  belonged to whatever directory the app was launched from, not to anything in the session —
  it now appears only when a project folder is actually attached.
- **The "model substitution" warning fires once per model pair, not on every request.** With a
  local llama.cpp model it logged the same warning on every single call (the server reports the
  loaded file's path as its model name), burying real warnings. A genuinely new substitution
  mid-session still warns immediately.
- **A local model is no longer reported as having been swapped for itself.** llama.cpp answers
  with the loaded file's path where other providers answer with a model name, so Bodega read
  "you asked for this model and got that file" as a substitution. It now recognises the two as
  the same model: no warning at all, and usage and cost are recorded against the model you
  picked instead of a file path that matches nothing. A provider that really does serve a
  different model is still reported, and still recorded as what actually ran.
- **The agent stops chasing things you mention in passing.** Telling it something was fixed
  elsewhere — in another repo, by another tool — could send it hunting through the current
  project for evidence that was never there, spending a dozen turns after the actual work was
  already done. Such remarks are now treated as context, not as a task.
- **The Context Inspector reports the tool set the model actually receives.** With deferred
  tool loading active it listed every registered tool (51) while the model's native set was the
  ~15-tool core — overstating both the tool list and the system-prompt token estimate. It now
  applies the same visibility filters as the real request path, deferred loading included.
- **The Explorer file tree fills its panel.** It stopped part-way down with empty space below
  and a scrollbar that appeared before it was needed, whenever the app was opened before a
  project.
- **Quieter logs.** Four spots trained you to ignore the log line that matters: an optional
  per-project config file logged a full error stack every time it was simply absent (a real
  read error on a config that DOES exist still logs, loudly); the Cloud Boost status check
  logged a line every 30 seconds even with Boost off, forever — it now logs only when the
  status actually changes; a webview navigation cancelled by a newer one landing first no
  longer warns (a navigation that's still current and genuinely fails still does); and the
  Context Inspector's 12-second background refresh only logs when what it assembled actually
  changed, not on every repeat. That same refresh was also quietly computing the compact-prompt
  cutoff as if every local model were cloud-hosted — display-only, now fixed.

- **Long local sessions no longer lose the original request when memory gets tight.** When Bodega
  summarizes older conversation history to make room, it asks your own local model to write the
  summary — on the same server that is already answering you. If that server was busy, the summary
  request timed out, and Bodega fell back to plainly cutting old messages instead. In one session
  that cut removed the request the whole task was about, and the final answer read like an answer to
  nothing. Now a busy model means Bodega waits and tries again on the next step rather than cutting
  anything, and if it ever does have to cut, your original request is kept.
- **"Couldn't summarize" no longer looks like "nothing to summarize" in the logs.** The two used to
  print the same line, so a session where summarizing never ran at all was indistinguishable from
  one where it ran and had nothing to do.

- **A forced answer now says so.** When the agent stops because it hit a limit — it stopped
  narrating and answered early, ran out of steps, or ran out of time — instead of the model
  deciding it was done, the response now carries a small note explaining why, so you can tell a
  considered answer from one that got cut off mid-thought. The note survives closing and reopening
  the session.

### Changed

- **Model Roles is simpler.** Settings → Models → My Models now shows four role pickers instead
  of eight — Default, Fast, Smart, and Agent, the ones that actually affect what answers you. The
  rest (Chat, Research, Debug, Advisor) moved behind an "Advanced: per-panel overrides" section you
  can open when you need them. Nothing you had assigned was changed or lost.
- **If your setup can only run one model at a time** (llama.cpp, LM Studio, KoboldCpp, GPT4All,
  MLX, TabbyAPI), the role pickers no longer pretend you can assign eight different models. You'll
  see one card that names the model you actually have loaded and explains that every role uses it.
  Previously you could fill in all eight pickers, hit Save, and nothing would happen — with no
  explanation why.
- **The VRAM warning now checks every role you've assigned**, not just Chat and Agent — so setting
  a different model for Fast, Smart, or an advanced override now gets the same "this might not fit
  your GPU" heads-up.
- **Responses now say which role answered.** When a message was handled by Fast, Smart, or Agent
  rather than your Default model, a small label next to the model name in the response shows which
  one served it.

## [1.0.0-beta.38] - 2026-08-27

### Added

- **Device frames are now device-true.** Real Dynamic Island, punch-hole and home-button chrome
  per phone, photoreal shells, and an OS status bar rendered inside the emulated safe area —
  content that respects the inset visibly clears the island, exactly like the real device.

- **New cloud models, live-verified 2026-08-27.** DeepSeek's `deepseek-v4-flash-vision-exp`
  (image input, priced identically to V4 Flash) and Z.ai's `glm-5.3-flash` (multimodal, 1M
  context) join the pickers, model profiles and pricing tables. GLM-5.3-Flash carries its 50%
  launch discount until September 9, then reverts to list price automatically.

- **Device-Frame Preview.** Preview a phone-sized build of your app: pick iPhone, iPad, Pixel or
  Galaxy in the Preview toolbar and the page renders at that device's real viewport, pixel density
  and user agent, inside a drawn device frame — with a rotate button. It is Chromium at device
  dimensions, not an iOS simulator, and the tooltip says so.
- **The Preview empty state can start your dev server for you** — one click runs your project's dev
  script in a Terminal tab and connects when the port answers.
- **Uncensored models in the catalog, opt-in.** An "Uncensored" filter in Discover (off by default)
  reveals community-modified models with reduced refusals — each entry states who modified it, what
  changed, and which claims are the author's rather than verified by Bodega. First entry:
  SuperQwen 3.8 27B Abliterated (vision, tools, checksum-verified download).
- **Mixture-of-experts models no longer show "doesn't fit" when your GPU and system RAM can carry
  them together** — the model card says "Fits with CPU offload" and hands you the exact flag to
  make it happen, with one-click apply. Bodega only claims a hybrid fit when it actually knows the
  model's expert layout; unknown models keep the plain VRAM verdict instead of a guess.
- **Machines with two or more GPUs get a multi-GPU editor in Advanced Flags** — per-GPU split
  proportions and a split-mode picker. Single-GPU machines see nothing new.

- **Eight new provider presets** (36 total): xAI direct (Grok without a router hop), Cerebras,
  SambaNova, DeepInfra, Novita, Nebius (EU-hosted), and two local runtimes — TabbyAPI (exllama-family)
  and SGLang. Each is a first-class preset with its base URL, auth expectations and health check
  wired in, not a "custom endpoint" you have to configure by hand.

- **Qwen3.8-Flash-Next recognized** (released 2026-08-26): the open-weight multimodal MoE preview of
  the Qwen4 architecture now resolves with its real profile — 262K native context, vision, thinking,
  ~180B MoE with 6B active — instead of falling back to 27B-dense assumptions. Honest limits: it is
  not offered as a managed local download (the smallest GGUF is 72.5 GB and the architecture is not
  in mainline llama.cpp yet), and the announced cloud id `qwen3.8-flash` is "available soon" with no
  reachable API, so no cloud entry or pricing ships until it can be verified live.

- **SuperQwen3.8-27b-abliterated recognized as a sideload**: the new full-BF16 refusal-edited
  Qwen3.8-27B (Apache-2.0, vision preserved, author-corrected overthinking) gets correct sampling and
  size when sideloaded, rather than inheriting the official 27B profile by filename accident. It is
  deliberately not a managed catalog entry — the author's claims are unmeasured by us.

### Fixed (wave 2, 2026-08-27)

- **MCP tools with whole-number parameters work now.** Any connected MCP tool that declared an
  integer parameter (most servers do — page sizes, limits, counts) had that parameter rejected no
  matter what the model passed, so models fell back to unbounded calls that flooded their context.
  Found live when a local model reading HQ channels couldn't cap "last N messages".

- **DeepSeek weekend calls now bill at the off-peak rate.** DeepSeek's peak-hour pricing applies
  Monday through Friday only; cost tracking previously charged the 2x peak rate on weekends too.
  Already-recorded costs are not restated.
- **GLM-5.1 pricing matches Z.ai's current card** ($1.40 in / $4.40 out per million tokens, with
  a cached-input rate — the old launch rate is no longer published), and GLM-5.2/5.3 gained their
  cached-input rates.

- **A stuck Windows GPU query can no longer freeze the whole app's VRAM detection.** One hung
  system call used to pin hardware info, model-fit checks and model launches behind a promise that
  never settled — everything now recovers within 10 seconds, and a one-off detection failure is no
  longer remembered as "0 GB VRAM" for the rest of the session.
- **Page errors now show up in the Preview itself** — a small errors strip with counts you can
  expand, instead of digging through DevTools.
- **Killed your dev server? The preview reconnects by itself** the moment the port answers again.
- **Pasting an external URL into Preview now tells you why it won't load** and offers to open it in
  the Agent Browser instead of failing silently.
- **The dev-only browser preview behaves like the real app when files are missing or operations
  fail** — file search, @-mentions and quick-open no longer break in browser preview.

### Security

- **Air-gap now also blocks chat requests to a non-local Ollama or Anthropic address** — previously
  only the OpenAI-compatible path was guarded at request time. It also skips the Cloud Boost health
  probe entirely at startup when air-gap is on, instead of dispatching the probe and refusing it after
  the fact.

### Fixed

- **A model that took the app down while loading is no longer loaded again on the next launch.**
  Bodega now records which model it is about to load and clears that record only once the model
  server is actually up. If the app (or the whole machine) goes down mid-load, the next launch does
  not walk straight back into the same load — it stays idle and tells you which model it skipped,
  with a button to try it again yourself.

- **The model-crash message says what happened instead of guessing.** Every crash used to end with
  "Try a smaller quant or check GPU memory", including crashes that had nothing to do with either —
  one turned up on a machine with 31 GB of free video memory and a 15 GB model. The message now
  quotes what the model server actually printed as it failed, says plainly when it printed nothing,
  and no longer offers a cause it does not know. An internal "Sleep aborted" that could reach the
  screen is gone too.

- **Downloaded models are checked against the publisher's checksum before they are usable.** A
  download that finishes at exactly the right size can still be wrong — a resumed transfer can stitch
  together bytes that do not belong together, and no size check can see it. Bodega now verifies the
  file's contents and discards it (with a clear message) if it does not match. Resuming a download
  also now only happens when the server can confirm the file has not changed underneath us; otherwise
  it restarts cleanly rather than risking a mismatched file.

- **A model file that is still downloading is no longer offered as loadable.** Dropping a GGUF into
  the models folder while it is still being written would index it at whatever size it had reached,
  so the picker listed a half-finished 15 GB model as ready to use. Bodega now waits until the file
  has stopped growing, and corrects the recorded size of any model whose file has changed since it
  was first seen.

- **Right-click menus now open downward from the cursor instead of upward over the panel above
  them.** Right-clicking a file near the top of the Explorer opened its menu growing upward, covering
  the panel header and the tab strip with the top of the menu cut off at the window edge. Every
  right-click menu in the app — file tree, editor tabs, terminal, diagnostics, sessions and projects
  — now opens below the click, flips above it only when there genuinely isn't room, and scrolls
  inside itself rather than running off the top or bottom of the window.

- **Context compaction no longer claims work it didn't do, and stops retrying a conversation it
  can't shrink.** On small context windows the log could say "compaction triggered" over and over
  while nothing was actually reclaimed — the line was written before the work, and the trigger kept
  re-firing on a conversation made entirely of recent messages it wasn't allowed to touch. The log
  now reports what was actually reclaimed, and after two back-to-back no-ops compaction waits until
  the conversation has genuinely grown before trying again.

- **A stuck local model now stops in seconds instead of silently burning the whole iteration
  budget.** When every tool call the model makes keeps getting held back by a safety guard (for
  example, it keeps re-reading instead of writing), the loop used to count each of those turns as
  progress and spin all the way to the iteration limit — up to a minute of nothing on screen, then
  the limit banner. It now notices a few of those no-op turns in a row, stops with what the model
  had so far, and says the run ended early.

- **Run traces now carry real numbers for local streaming calls.** Every streaming LLM call used to
  log zero tokens and zero latency, which made a stuck run look identical to a healthy one in the
  diagnostics export. Latency is now the measured call duration, and token counts fall back to an
  estimate when the local server doesn't report them.

- **The editor's active line is no longer bright red.** The theme handed the editor a colour format
  it cannot parse, and its fallback for that is literal red. Every theme colour is now converted to a
  form the editor accepts, so this whole class of accident cannot recur.
- **The context inspector has a solid background again** instead of letting the panel behind it bleed
  through.
- **Code Mode's context sheet now dims the panel behind it** (click the dimmed area to close). Without
  the dim, streaming text touched the sheet's top edge and read as bleeding through it.
- **The "Active" badge on Settings → Prompts no longer gets cut off by the card edge.** The
  Code and Chat default rows stopped shrinking to fit once one of them held a long option name, so
  at narrower window widths the second badge was pushed past the edge of the card.

- **Commit messages in the Source Control history fill the available width** instead of truncating
  after a few characters however wide the panel was.
- **Your OpenAI API key no longer travels to servers that are not OpenAI.** If `OPENAI_API_KEY` was
  set in your environment, every OpenAI-compatible provider — a custom endpoint, LM Studio, vLLM —
  silently used it as its auth token, sending it to whatever base URL was configured. The environment
  fallback now applies only to OpenAI's own API; other endpoints use only the key you typed for them.
- **A custom OpenAI-compatible endpoint without an API key works again.** Auth is optional for the
  Custom provider, but part of the app treated a missing key as "not configured", so chat said the
  custom API was not set and the model picker showed it as unreachable while Settings looked correct.
- **Opening the model picker or checking a provider's health under Air-Gap mode no longer makes a
  real network call.** Everything that actually sends a chat message was already blocked; two
  background checks — refreshing the model list and the provider health badge — weren't, so a
  non-loopback endpoint (a custom API, a LAN server) could be probed over the network while Air-Gap
  was on. Both now refuse the same way the rest of the app does, and fail quietly instead of erroring.
- **MiniMax's health badge no longer shows "unhealthy" for a working key.** MiniMax has no `/models`
  endpoint to check against, and the health check was always using one anyway — it now checks the
  endpoint each provider actually has.
- **A saved Ollama URL takes effect immediately** instead of only after restarting the app. Changing
  it (or an Azure resource name) used to update Settings but leave the app talking to the old address.
- **Provider connection errors say what's actually wrong.** "Local server unreachable" no longer shows
  up for a Custom or Remote endpoint that isn't local by design — those now point at the base URL and
  key instead. A cloud provider returning zero models now distinguishes a rejected API key from a
  connection problem instead of always saying the same generic thing.
- **vLLM now gets its repetition-loop-prevention sampler.** It never reached vLLM under either its
  own field name or llama.cpp's, so a vLLM-served model had no defense against getting stuck
  repeating itself — every other local backend already had this protection.
- **Spending shows "no price data" instead of a made-up number for Concentrate and Featherless.**
  Both used to bill every call at the same flat guess regardless of which model was actually used —
  now a call with no real price on file is labeled honestly rather than presented as an accurate
  charge. Z.ai was already priced correctly for its GLM models; a stale comment overclaiming this
  for Concentrate was corrected too.
- **Streamed responses through MiniMax and Concentrate now report a cost** instead of silently
  showing nothing. Neither provider sends the usage data streaming responses need, so the app now
  estimates it from the response length rather than reporting zero.
- **A corrupted usage report from a streaming provider no longer gets treated as "this response cost
  nothing."** A malformed accounting frame used to look like a confident zero; it now falls back to
  an estimate instead.

## [1.0.0-beta.37] - 2026-08-23

_V2 hardening pass (2026-08-23) — no new features, just making what's already here work the way it
was supposed to. Fixes, behaviour changes and speed-ups below are from that pass unless noted
otherwise; each lane appends its own lines under the matching heading._

### Added

- **The agent browser reads a page as a short, labeled list of clickable things instead of raw
  HTML.** A new `getSnapshot` action lists links, buttons and fields with short refs (`e3`), flags
  hidden elements, and is a few hundred tokens instead of eight thousand characters; the agent can
  then click or type by ref instead of guessing a CSS selector, and a ref from an old snapshot is
  refused rather than silently pointed at something else. Page text now reaches the model inside one
  clearly marked untrusted block, and `getDom` drops scripts and styles before its size cap.
- **The agent's browser can press keys, choose dropdown options, hover, scroll, and go back or
  forward** — gated exactly like clicking, and back/forward never move to a page that would have
  been refused as a navigate.
- **An "Uncensored" prompt template** (Settings → Prompts) for models you chose because they
  answer without refusals. It drops Bodega's own "decline harmful requests" instruction — which was
  making such models refuse — and keeps truthfulness and privacy. Not the default; pick it as the
  Chat default when running one of those models. Existing installs get it on next start.
- **Rename Symbol (F2)** now works in the code editor — it was wired for Go to Definition,
  Find References, Hover and Signature Help but not renaming. Renaming updates every open file
  the rename touches; a file with unsaved changes is left alone rather than overwritten. Real
  quick-fixes from the language server (add a missing import, etc.) now show up alongside the
  "Fix with Bodega" action in the lightbulb menu. The Symbol Outline panel and the new **Go to
  Symbol** overlay (**Ctrl+Shift+O**, jump to a function/class/etc. in the current file) now read
  from the language server when one is available, falling back to the previous best-effort
  scan for languages without one.
- **The agent browser can now wait for something to happen instead of guessing.** A new
  `wait_for` action polls for a selector, some page text, a URL substring, or the page settling —
  useful for pages that load content after the initial page appears. Clicking a link or button now
  reports whether the click actually navigated, is still loading, or stayed on the same page,
  instead of leaving you to check separately. A navigate that runs out of time now double-checks
  whether the page landed anyway before calling it a failure, so a slow-but-working page is no
  longer reported as broken.

### Changed

- **Agent-browser site approvals survive an app restart** (per project, in the local database), and a
  fixed list of banks, payment processors, brokerages, government-ID portals and adult sites can never be
  automated by the agent — refused before any approval card, with no setting to unlock it.
- **Typing and submitting forms on your own localhost dev server no longer stops to ask for
  approval on every keystroke and click** — password, credit-card and one-time-code fields still
  ask every time. Agent-browser screenshots sent to the model are now resized to a reasonable
  size by default, so a big monitor's screenshot no longer eats your whole prompt; add
  `fullRes:true` if you need the bigger picture.
- **One hunk-review workflow, not two.** The batch review panel (`Ctrl+Shift+D`) and the editor's
  in-tab Diff Review now share one keyboard set (`j`/`k` move, `a`/`r` accept/reject, `Ctrl+Enter` apply,
  `Esc` close), and a new "Open in Diff Review" button carries your accept/reject choices from the
  batch view into the in-tab view instead of resetting them.
- Fleet monitor rows use the same table rows and cells as every other table in the app (spacing,
  hairlines and number alignment now match).

- **A quieter surface.** The app's panels no longer float as separate cards with their own
  shadows; everything sits on one surface divided by thin lines, and the selected item in lists
  and menus is shown with one soft highlight instead of a purple bar or fill. The light theme is
  a softer warm grey rather than near-white, so switching to it at night no longer glares. The
  code editor and terminal pick up the same colours and the same light/dark syntax palette, and
  they re-colour immediately when you change theme. The fleet cards' coloured gradients are gone.
- **One set of building blocks.** Dropdowns, context menus, tooltips, inputs, toggles,
  badges, banners, tab strips, status dots and empty states across the app now come from a
  single shared set, so they look and behave the same everywhere — the same menu density,
  the same keyboard navigation, the same quiet highlight for the selected row. Buttons are
  text or outlined, never filled. Menus and panels no longer each pick their own stacking
  order or shadow.
- **Chat Mode, reorganized.** The app is one surface now — no floating panels with gaps and
  shadows, just thin dividers between the sidebar, the conversation and the side panels. The
  sidebar lists sessions as plain rows with a soft highlight on the current one (the purple bar
  is gone), and starts a new chat from an outlined button. Replies no longer sit in a box; the
  agent mark and name introduce them and the text reads on the page. Your own messages are a
  quiet grey bubble. Tool calls are one-line rows, and an approval is a small card with Reject /
  Allow and their keyboard hints. The composer keeps its controls inside the box — attach,
  Thinking, Web, the model, mic and send — and the old strip of chips below it is gone: Research
  and routing mode live under "+", the context inspector has its own button at the top of the
  conversation. The top bar's Chat/Code switch is a quiet pill instead of an underline.

- **The editor's Diff Review tab can now keep or drop individual hunks**, not just the whole file.
  Whole-file Keep/Undo still work exactly as before; toggling is optional — every hunk starts
  accepted. Use `j`/`k` to move between hunks and `a`/`r` to accept/reject the focused one, or click
  Accept/Reject on any hunk directly. A "N of M hunks kept" line tracks your selection.

_(Calm Surface · Wave 4 — Settings, first run, overlays, help)_

- **Settings, first-run setup, dialogs and the help hub now match the rest of the app.** The
  Settings nav uses the same quiet highlight as everywhere else (no purple bar), theme cards pick
  with a hairline and a check instead of a tint, spending ranges are a segmented control (the
  filled "30d" is gone), model and provider cards, MCP/ACP/plugin/skill rows, hook lists and
  knowledge cards all use the shared rows, cards, badges and meters. The first-run flow and
  provider/model setup steps read as the same app — text step progress, hairline cards, no filled
  buttons — and the trust prompt, confirm dialogs, What's New, keyboard shortcuts and command
  palette sit on one modal/popover recipe with a theme-aware scrim. Help hub pages use the
  reading typography of the chat stream; the file viewer centres its content column.
- **`text-error`/`border-error` classes referenced a colour that was never defined** — several
  llama.cpp error states rendered with no colour at all. Fixed.
- **Two doc-hub sentences still described purple-filled toggles** (Research, Boost). Corrected.

_(Calm Surface · Wave 3 — Code Mode)_

- **Code Mode now sits on the same calm surface as Chat.** Explorer rows, editor tabs, the Agent
  panel, Sessions drawer, Fleet cards, Git/Search/Todo/Ports/Problems panels, terminal overlays,
  preview bar and the codebase Map all use tonal selection (no purple rails or fills), hairline
  panel headers, the seven-step type scale and the shared Row/Badge/Card/Tabs/EmptyState/Meter
  pieces. Fleet cards lost their per-kind glow; the selected file is a quiet highlight, not a bar.
- **The Web toggle is gone from the Code Mode composer.** The agent's web search and fetch tools
  are available whenever tools are — the toggle only ever changed a line of prompt text, and with
  it off that line told the model it *couldn't* search even though it could. Just ask. Chat Mode
  keeps its Web chip.

_(Code Mode editor)_

- **Files opened outside your project folder now open read-only**, with a badge, instead of
  silently accepting edits you couldn't have saved as part of the project.
- **Added Ctrl+=, Ctrl+-, and Ctrl+0 to zoom the editor's font size** in and out and reset it,
  without leaving the keyboard for the Settings panel.
- **The editor can now split vertically (stacked) as well as side-by-side.** A toggle next to the
  split button switches orientation.
- **Diff review now has keyboard bindings**: Ctrl+Enter to Keep, Ctrl+Backspace to Undo, Esc to
  Close, in addition to the existing buttons.
- **Hidden tabs are now reachable from a new overflow menu** when the tab strip scrolls past the
  visible width, instead of only the scrollbar.
- **The status bar now shows line-ending (LF/CRLF) and encoding for the active file.**
- **Whitespace rendering is now a Settings choice** (Selection / All / Boundary / None) instead of
  being fixed to "only while selecting text."
- **Opening a large file (5–20MB) now asks before loading, from every entry point** — Quick Open
  and go-to-definition used to open one instantly with no warning; the file tree already asked.

### Fixed

- A project with Air-Gap Vault on (global air-gap off) no longer probes a non-local Ollama endpoint when
  choosing a vision model; the probe now honours the vault like the image dispatch already did.
- **The chat transcript now shows the model that actually answered** when the local server serves a
  different model than requested — for interactive and background/headless runs alike.

- **Chat Mode's Agent Browser panel now shows running local dev servers**, matching Code Mode.

- **Keyboard navigation in seven right-click and dropdown menus** (file tree, chat session list,
  problems panel, editor tabs, agent browser options, terminal, and Run Tasks) — arrow keys, Home
  and End now move between items again, and Enter activates the highlighted one. Mouse-only use
  still works exactly as before.
- **Two overlapping labels on the API key field** when connecting a cloud provider (Settings →
  Providers), which could confuse a screen reader about which text actually names the field.
- **Tab strips could skip the wrong tab** when arrow-keying past a disabled one with keyboard
  navigation, landing back on the tab you started from instead of wrapping around.
- **Chat Mode now tells the agent about every tool it can actually use.** The tool list shown to
  the model in Chat Mode was quietly narrower than what it was actually allowed to call — file
  search, file reading, and the browser preview tools worked in Chat Mode but were never mentioned
  in the model's own instructions, so it sometimes acted as if they didn't exist. Both now agree.
- **Searching your saved memories for a term containing a percent sign or underscore** now finds
  only what you typed, instead of treating those characters as wildcards and returning
  unrelated results.
- **Editor tabs were mouse-only for reordering and the right-click menu.** Focus a tab and press
  **Alt+←/→** to move it, or **Menu** / **Shift+F10** to open its context menu (pin, close, close
  others) — the same actions dragging and right-clicking already had. Tabs also now announce their
  name, position and unsaved state to screen readers.
- **Go to Symbol (Ctrl+Shift+O) only matched an exact substring** — typing "gts" for
  "GoToSymbol" found nothing. It now fuzzy-matches like Quick File Open, and each result shows a
  muted kind marker (function, class, etc.).
- **Breadcrumbs showed the file path only.** They now add the enclosing function/class at the
  cursor's current position, e.g. `file.ts › handleSubmit`.

_Bugs found and closed during the V2 hardening pass, plus carried-over live-test fixes from the same
window._

- **The app said "No model loaded" after start-up even though llama.cpp had loaded it.** The
  window only re-checked every 30 seconds, skipped the check while anything was streaming, and
  never listened for the server's own "model ready" signal. It now listens for that signal,
  checks a few times quickly right after start-up, and re-checks the moment a reply finishes.
- **A local model could loop inside its reasoning for minutes.** The loop detector only looked
  at the visible answer, and only after the reply ended. It now watches the reasoning and the
  answer while they stream and stops a runaway loop within a few words.
- **Local models now get a sensible sampling floor** (repeat penalty 1.1, min-p 0.05, top-p
  0.95, top-k 20) when you haven't set your own values — the usual guard against repetition on
  quantised models. Your own settings, per-model overrides, Ollama and cloud providers are
  untouched. Noted in Settings → Models.
- **That sampling floor now follows the model actually serving your request, not just the app's
  global preset.** Running the CLI against a local llama.cpp server while the app's own preset
  is set to something else previously skipped the floor entirely; a Cloud Boost run over a local
  base preset could have wrongly received it. Both now match the model doing the work.
- **Closing the floating Sidechat window no longer cancels a run in progress.** It used to abort
  the turn the instant the window lost its connection, the same way a stray tab-away used to
  cancel a main chat before that was fixed — a Sidechat answer now keeps generating in the
  background and is there when you reopen it.
- **A sideloaded GGUF whose filename merely contained a catalog model's name inherited that
  model's reasoning-depth contract.** Reasoning tiers now apply only to models downloaded from the
  catalog.
- **A local model with unreliable tool-calling could burn many extra tool calls stuck on a tiny
  mistake before Bodega gave up and asked it to stop.** Bodega already gave capable models a
  little more rope before intervening; it now gives less-reliable local models a little less,
  instead of treating every model the same once it's doing agent work. Well-calibrated cloud and
  large local models are unaffected.
- **Bodega mirrored a hostile tone back.** The style-matching guideline now matches how much
  you write, not how you treat it.
- **Code Mode's context ring never filled or opened anything** — the budget was being stored
  against the Chat session. Fixed.
- **"Code-only in practice" was clipped in the Settings nav** — it reads "Code only" with the
  full note on hover.
- **The reasoning block vanished once a reply finished.** It now stays, collapsed, above the
  answer; the live block is capped in height and scrolls.
- **llama.cpp came up with no model on every cold start, even with a model selected.** After
  each load, Bodega rewrote the selected-model setting to the server's full file path, which the
  start-up check could not match. Path-shaped values now resolve, and the rewrite no longer
  happens.
- **Loading a model from Settings briefly killed it and loaded it again.** The settings save
  behind the click queued a second load of the same file; it now joins the one in flight.
- **The repeat penalty in the local sampling floor never reached llama.cpp.** It was being
  dropped on the way out; it is now sent, for llama.cpp only.
- **Qwen 3.8 running locally used the global temperature (0.4)**, which is the known repetition
  trap for dense Qwen 3 models. It now gets the family's recommended 0.7 unless you set your own.
- **The loop detector missed numbered loops** ("579. Filter / 580. Add a solution / …") because
  the number changes every line. It now also compares lines with the numbers stripped.
- **Chat Mode answers from a local model were acknowledgements instead of content** ("Alright,
  here's the rundown." and then nothing). Every chat turn carried the full list of Code Mode
  skills, and a mid-size local model spent its reasoning sorting your question into them. Chat
  can't run skills, so the list no longer goes there; slash commands still work.
- **"Quick Questions — what programming language?" popped up for an ordinary chat question.**
  "Teach me how to make X, step by step" was read as a vague coding task. The interview now only
  runs on turns that can actually create files.

### Performance

_Nothing feels slower, and a few things are measurably faster or lighter._

- **The llama.cpp status poll runs only when llama.cpp is your provider** — it used to ask the
  server for its status every second on every provider, forever.
- **Language-server diagnostics no longer pile up for the life of the session** — they're dropped
  when a file closes, the server goes away, or the client is disposed.
- **Long agent runs no longer keep an unbounded transition log** (capped; the lifetime count is
  still reported).
- **Rate-limit pacing on OpenAI-compatible providers stopped over-counting your own system prompt
  every turn**, which was adding long, needless waits between turns on small-tier keys.
- **Memory check, for the record:** the backend holds ~590 MB idle and stays flat (+4 MB) across
  20 chat turns, an agent run, and ten idle minutes — no leak.

## [1.0.0-beta.36] - 2026-08-20

### Security

- **The agent browser's cookie isolation is now proven, not just asserted.** A new automated test
  launches a real copy of the app's browser engine and checks, directly, that a cookie set while you
  browse normally never reaches the agent's separate browsing session, and a cookie the agent picks
  up never reaches yours. Previously this was only checked with a stand-in for the browser engine,
  which could not actually prove the two stayed separate.
- **A setting could be used to run commands on your machine.** The custom SSH
  key path in Git settings was passed to git in a way git interprets as a
  command line, so a crafted value could execute anything the app can execute —
  and because the agent can change settings, a prompt-injection attack on a
  hostile web page or repository could reach it. Key paths are now validated and
  rejected outright if they are not a plain path to a real file, and a rejected
  key tells you instead of quietly falling back to your default SSH key. This
  protection now also covers pushes the app makes on its own — automated
  pull-request pushes and similar background actions — not only pushes you
  trigger by hand.
- **Approving or revoking trust for a project's own skills is now tied to your
  actual permission mode.** The approve and revoke actions had no connection to
  Ask/Plan/Act at all, so anything running locally — including a command the
  agent itself had been allowed to run — could grant or remove trust in a
  skill while you were in a mode whose entire point is that nothing changes
  without you. Both actions now respect your permission mode, and every grant
  is now recorded with what was approved, when, and how it was requested.
- **A skill folder that was actually a shortcut to somewhere else on disk
  could load and run without ever showing up for approval.** A project's
  skills folder could contain a filesystem junction pointing outside the
  project; the loader followed it and ran the skill normally, while the
  approval list showed nothing waiting — the exact protection this feature
  exists to provide, defeated by a filesystem trick a cloned repository could
  ship. These are now refused outright and shown as blocked.
- **Creating a skill or importing a plugin could bypass Ask/Plan mode
  entirely.** Whether the action needed your approval depended on a value the
  request itself supplied, so a request that simply left that value out wrote
  directly no matter what mode you were actually in. Both now always check
  your real, saved permission mode — and the same gap, one layer down, in how
  a tool call's approval is decided during a chat, is closed with it.
- **The built-in browser could send a click or a typed value to a live,
  non-local site while a project was marked air-gapped.** Navigating and
  submitting a form were already re-checked against air-gap status; clicking
  and typing on an already-open page were not. Reading a page you already have
  open — viewing it, screenshotting it, checking for console errors — is
  intentionally still allowed under air-gap, since that sends nothing off your
  machine; only the actions that can send data out are blocked.
- **Content from a vaulted project could be pulled into a different,
  non-vaulted project's cloud request.** Searching across all your projects at
  once is intended and unchanged; only rows that came from a project you
  marked as vaulted are now kept out of a search made from a different,
  non-vaulted project's session.
- **Reading files was not confined to your workspace.** Writing, deleting and
  renaming were restricted to the folders you had opened; reading was not, so a
  compromised page in the built-in browser could ask for any file on the disk.
  Reads are now restricted the same way. Files you pick yourself through the
  system file dialog still open normally.
- **Air-gap mode could be defeated by an unrelated program.** The app confirmed
  air-gap status by asking its own backend over the network, and treated any
  answer on that port as authoritative — so a leftover backend from an earlier
  session, or any other program answering there, could report air-gap as off
  while it was on. The setting on disk is now read first and always wins.

### Fixed

- **Starting a Fleet Parallel run on a repo with no commits yet now shows a
  clear message instead of raw git error text.** A fresh `git init` with no
  commits made "git rev-parse HEAD exited 128: fatal: ambiguous argument
  'HEAD'..." bubble straight into the run's error state. It's detected up
  front now, before anything is provisioned, and tells you plainly: make an
  initial commit first.
- **The Bodega Map's rebuild button now tells you what it actually found.**
  A rebuild that changed nothing looked identical to a broken one — both just
  reset the "built 0s ago" label. The Map graphs your project's structure
  (files and imports), so a content-only edit correctly leaves it unchanged;
  a rebuild now says so directly ("rescanned N files — no structural
  changes") or tells you what moved ("rescanned — N nodes changed").
- **A panel that crashed from a rebuild happening underneath it now tells you
  to reload, instead of offering a "Try Again" that could never work.** If the
  app's files changed while it was running — a rebuild, an update landing
  mid-session — a panel like the Code Editor could crash trying to load a
  piece of it that no longer existed on disk. "Try Again" rebuilt the panel
  and asked for the exact same missing piece, so it failed identically every
  time; only a full app reload actually recovered. That specific crash is now
  recognized for what it is, and the panel offers Reload App with an
  explanation instead of a retry button that was never going to succeed.
- **VRAM fit estimates for vision models now include the projector.** The
  green "fits" badge and the predictive VRAM check both compared a vision
  model's weights alone against your available VRAM, leaving out the
  multimodal projector file (mmproj) every vision model also needs — 0.16 to
  1.9 GB depending on the model. A card could read as fitting and still run
  out of memory once the projector loaded. Both now count the projector,
  using its real downloaded size when it is already on disk and the
  documented catalog estimate otherwise. Non-vision models are unaffected.
- **Turning reasoning "Off" did nothing on some models.** The composer
  discarded your choice before it reached the model, so picking Off produced
  the exact same request as leaving reasoning on its default — on DeepSeek
  and thinking-capable Qwen models, that meant you were billed for reasoning
  you had explicitly turned off. Off is now actually sent as Off, and the
  in-app documentation, which claimed reasoning was off by default, has been
  corrected to say what the app actually does.
- **Behind-the-scenes groundwork for reasoning controls that only ever offer a
  position your model actually supports.** OpenAI, Gemini, DeepSeek and
  Qwen/DashScope now each carry a data-driven description of exactly which
  reasoning depths they accept and how they behave when asked for something
  they don't — DeepSeek's "Medium" is recorded as the same request as "High"
  it's always been, rather than a separate option that happened to collapse
  silently. This groundwork doesn't change what you see in the composer yet;
  it's the foundation the reasoning picker will read from next.
- **Qwen 3.8 27B's Low / Medium / Extra High picker now actually reaches the
  model.** The picker itself started showing real levels in an earlier
  release, but the value you chose never made it into the request the local
  server received — every turn ran at the model's most expensive default
  regardless of what was selected. It's now sent on every turn, mapped to the
  model's own vocabulary.
- **The "Off" row on the reasoning picker no longer shows for a model whose
  provider has no way to actually turn reasoning off**, for every provider —
  previously this protection only covered Anthropic's always-thinking Claude
  models; OpenAI- and Gemini-family models with the same limitation could show
  an Off option that silently did nothing when picked.
- The reasoning picker now reads its options from what the backend confirms a
  model actually supports, verified end-to-end against what the server
  serves — not only against hand-written test fixtures — closing the last gap
  in last release's move away from name-guessing.
- A leftover, never-taken code path for the legacy "inject reasoning into the
  prompt" fallback has been removed; the fallback itself is unchanged and
  still covers models with no native reasoning parameter at all.
- **Adding a web page to your knowledge base from a second project could
  silently delete it — and everything saved from it — out of the first
  project.** Re-adding a URL you'd already saved elsewhere removed the
  existing entry without checking which project it belonged to, with no
  warning and no way to undo it. Re-adding a URL now only replaces that same
  URL's entry within the project you added it from.
- **Undoing a single remembered fact through the agent's own revert could
  delete the same fact from every other project, and the general one, along
  with it.** Reverting one memory now only reverts that one memory.
- **Turning on semantic memory search after you already had memories saved
  made every older memory invisible to search, with nothing telling you it
  had happened.** Enabling embeddings only ever helped memories saved from
  then on; the moment one embedded memory existed, everything saved earlier
  silently stopped showing up in search results, even though it was still on
  disk. Search now finds memories from before and after you turn embeddings
  on.
- **Claude usage under-counted cost on any conversation that reused cached
  context** — which is most agentic sessions, since Bodega's own working
  context is exactly what gets cached. Cached tokens were being left out of
  the calculation entirely instead of billed at their real (lower) rate.
  Recorded cost on Claude turns will look higher after this update — that is
  the number becoming accurate, not a price change; nothing about what
  Anthropic actually charges has changed.
- **A model your budget didn't recognize could spend past your cap with
  nothing to stop it.** An unpriced model was estimated at $0 per token, and
  a $0 estimate skipped the daily-budget check entirely — so spend on that
  model was never blocked, no matter how low your limit was set. Unpriced
  models are now estimated conservatively above their likely real cost
  instead of at zero, so a budget cap actually catches them.
- **Several models were billed at the wrong rate.** o3 was charged at o1's rate,
  7.5x too high. Opus 4.5 was charged at the pre-4.5 rate, 3x too high, in Boost
  usage. GPT-5 was charged at GPT-5.4's input rate. `deepseek-chat` was charged
  as an older model than the one that actually answers it. mistral-large was 4x
  too high and one GPT-5.6 tier 5x too high. Every rate is now checked against
  the provider's own published pricing, and the app's three pricing tables are
  checked against each other automatically, so they cannot drift apart again.
- **Claude Sonnet 5 is billed at its introductory rate.** The published $2/$10
  promotional rate runs to 31 August; the app was charging the standard $3/$15.
  The change-over back to standard pricing is now dated, so it happens on its own.
- **DeepSeek peak and off-peak pricing starts at the right hour.** DeepSeek's new
  time-based pricing begins at 16:00 UTC; the app switched at 00:00 UTC, so usage
  on 16 August was costed against the wrong rate for sixteen hours.
- **Closing a tab could approve a tool by itself.** A pending approval left on a
  closed window kept its countdown running, so a read-only tool could approve
  itself and run with nobody watching, and a longer-running one could be
  cancelled minutes after you walked away. Pending approvals now wait for a real
  decision, and the run itself keeps going as before.
- **The app could learn you had refused something you never saw.** A disconnect
  was recorded as if you had pressed Deny, and one such record permanently
  prevented that action from ever being suggested for automatic approval again.
  Disconnects are no longer treated as decisions.
- **A dropped connection could lose a tool call and still report success.** If a
  connection ended before the provider sent its end-of-stream marker, a completed
  tool call was discarded and the turn finished as if the model had simply chosen
  not to use a tool — no error, nothing in the logs. Affected OpenAI-compatible
  providers including OpenRouter, Groq, vLLM and LM Studio.
- **Cut-off replies were shown as complete.** The backend marked a response as
  truncated and the interface ignored the flag, so a partial answer looked
  finished — and for hosted models that signal was also used to decide the model
  was warmed up and working.
- **Retrying a too-long conversation lost your reasoning settings.** When a
  request was retried after exceeding the context limit, "Thinking: Off", fast
  mode and custom sampler settings were dropped from the retry, so models that
  think by default started thinking again despite being turned off.
- **Prompt-cache statistics read zero after a reconnect**, which looked like
  caching had stopped working when it had not. Approval prompts shown after a
  reconnect also displayed internal tool identifiers instead of names.
- **A local model could keep answering after you switched models.** If shutting
  down a local server failed, the app lost track of it while it kept running and
  answering on the same port, so the next model appeared to load successfully
  while the previous one served every reply. The app now confirms a shutdown
  before starting a replacement and checks that the server answering is the one
  it started.
- **A missing tool was reported as broken code.** On Linux, a verification
  command failing because the program was not installed — Python, for example —
  was read as a genuine compilation error, and the agent was told to repair code
  that was fine.
- **Custom `--parallel` settings for llama.cpp no longer break the context
  budget.** Setting it split the context across slots without the app knowing, so
  it planned prompts against several times the space actually available and
  requests were cut off mid-run.
- **Skills and tools no longer appear briefly empty at start-up.** Both lists
  could be requested before they had loaded and would return an empty list as if
  that were the answer.
- **External coding agents (ACP client mode) are now clearly marked
  unavailable, with a reason, instead of appearing installed and then hanging
  on the first message.** The agent picker showed an external agent as ready
  the moment its program was found on your machine, even though a real
  conversation with it could never finish. This mode is off by default while
  it is rebuilt against the actual protocol those agents speak; the picker
  now says so instead of letting you start a conversation that cannot end.
- **Cancelling a slow MCP tool call no longer disconnects the whole MCP
  server.** A single tool call running past its timeout was treated the same
  as the server itself dying — every other call in flight to that server
  failed too, and the server was reconnected even though it was healthy the
  whole time. A timed-out call is now reported on its own, and a healthy
  connection is left alone.
- **A wedged run in Loops now releases its slot on its own instead of
  blocking every other scheduled and overnight run indefinitely.** A run
  whose provider stopped responding without closing the connection used to
  sit "running" forever, permanently occupying the one concurrent run slot
  and making every other loop report a false concurrency error until you
  restarted the app. Loop runs now have a maximum duration (30 minutes by
  default) after which a stuck run is stopped and marked failed, clearly
  labeled as a duration timeout rather than a cancellation.
- **An apply interrupted by a crash or a restart no longer leaves a run stuck
  on "Applying..." forever.** That state could previously only be cleared by
  restarting the app, and even then it stayed stuck. Bodega One now recovers
  an interrupted apply automatically on restart, and marks it as failed if it
  didn't finish, so it can be retried, completed, or discarded normally.
- **A code-intelligence server (language server) that keeps crashing right
  after it starts is now retired and replaced instead of endlessly
  restarting or going silent forever.** A server that OOM'd on a particular
  file used to either respawn on every keystroke with no back-off and no
  visible error, or — for the agent's own use of the same servers — go quiet
  permanently after a couple of failures with no diagnostics and no
  indication anything was wrong. Both cases now surface a clear failure and
  recover with a fresh server automatically.
- **Approving or revoking a project skill's trust no longer freezes every
  open chat while it happens.** The trust store read and wrote its file
  synchronously, which briefly stalls the entire app — including any chat
  that is mid-response — and could take over twenty seconds on a busy disk.
  Trust changes are now handled without blocking anything else you're doing.

- **Connecting an external coding agent (Claude Code, Gemini CLI, Codex,
  Cursor) to Bodega, or connecting an editor like Zed to Bodega, no longer
  hangs on the first message.** Both directions of this connection were
  speaking a protocol invented before either side had been checked against
  a real agent or a real editor, so a genuine connection stalled instead of
  completing its handshake. Both directions have been rebuilt to match the
  actual protocol these tools use. External agents stay off by default until
  a real one has been verified end-to-end; turn them on with
  `acp.client_mode_enabled` if you want to try them early.
- **Cancelling an in-progress turn from an external agent connection now
  reports the cancellation correctly** instead of through a made-up signal
  that no real client or agent recognizes.
- **A crashed language server used to keep showing green.** If the background
  helper that powers go-to-definition, hover info and error squiggles for a
  language crashed or was stopped, the editor kept its old status badge and
  kept showing outdated errors until you restarted the app. A crash or stop
  is now reflected immediately — the status badge updates and stale error
  squiggles clear.
- **Inline code completion silently went quiet when it hit your spend cap or
  had no active model, and looked identical to just not typing anything worth
  completing.** Both cases are now told apart in the completion stats, and a
  cancelled completion request is now actually cancelled instead of finishing
  anyway after you had moved on.
- **A scheduled or batch run that lost track of its own commit, or was set to
  run as an agent profile that had since been deleted, used to fail
  silently.** A merge that landed but could not auto-commit now shows a
  warning on the run instead of only in a server log; a run configured for a
  deleted agent profile now says so on the run instead of quietly falling
  back to default settings.
- **Deleting an agent profile that a scheduled loop still points at now warns
  you and names the loop**, instead of deleting silently and leaving the loop
  referencing nothing. You can still force the delete if you mean it.
- **A batch of runs you cancelled partway through could show an item as
  finished when it never ran.** Items still in flight when you cancel now
  show as skipped, not done.
- **Bodega Map's file and edge counts are now honest about the cap** — when a
  project has more files than the map can show, the stats say so instead of
  quietly freezing, and edge counts are split into import edges and
  structural edges instead of one combined number. Refresh now actually
  rescans instead of sometimes returning the same cached map.
- **A tool-approval card for an MCP server now names the server**, not its
  internal id, so you know what you're actually approving.
- **Memory context size shown in stats is now measured from what was
  actually injected into the conversation, not estimated from a fixed
  multiplier** that stayed the same even when nothing was injected.
- **Some llama.cpp flag decisions used to happen silently** — a
  chat-template flag being dropped because the model doesn't support it, or
  a global context-size override being ignored in favor of a per-model one.
  Both now show up in the debug frame instead of leaving you to guess why a
  setting didn't seem to take effect. A per-model context-window setting now
  correctly takes priority over a global one, matching how every other
  per-model override in the app already works.
- **A scheduled run blocked by a failing spend-cap check now says so instead
  of silently proceeding as if nothing were wrong.** A vault or spend-cap
  lookup that itself fails now blocks the run rather than treating the
  failure as permission to spend.

- Several internal documentation comments that promised behavior the code did not deliver have been corrected — including the repo map's cache-refresh claims and the map query tool's air-gap status, which now states plainly what is and is not enforced.
- Help-page references to source locations were repaired after code moved underneath them.

### Added

- **The reasoning-effort picker on Qwen 3.8 27B (local) now shows its real
  Low / Medium / Extra High levels** instead of a plain on/off toggle. Bodega
  now sends the model's own effort value instead of always leaving it at its
  most expensive default.
- **There's finally a way to set a default reasoning effort for every model
  at once.** Settings → Models → My Models → Model Roles gains a Reasoning
  Effort control under Global — previously this setting existed and was
  documented, but nothing in the app could actually change it.
- **The thinking toggle now appears only for models that actually support it,
  and the same is true for the effort picker.** Previously a stale menu row
  could sit there for a model whose provider rejects it outright — most
  visibly, the strongest Claude models refuse to turn thinking off at all, so
  showing an Off option there did nothing when you picked it. That option is
  gone for those models now; every model with tunable reasoning still shows
  the full effort picker. Claude's reasoning replies are also more consistently
  populated instead of occasionally showing an empty "thinking" section.
- **You can now revoke a project skill's trust, not just grant it.** Trust
  could previously only be given; taking it back required editing a file by
  hand. Settings gains a revoke action alongside approve; the command-line
  tool's approve command gets the same fields it needs to keep working in a
  future release (see below).
- **The agent can open pull requests.** A dedicated tool with proper checks,
  rather than shell commands: it always asks first — in every mode, including
  when other actions run automatically — and refuses to open a pull request from
  a protected branch.
- **Commits made by the agent say so.** Agent-authored commits carry a trailer
  identifying the model, alongside the existing signed verification trailer used
  for verified runs. It is on by default and can be turned off in settings.
- **Pushing to `main` or `master` is refused**, along with the repository's own
  default branch, whatever it is called. Force-pushing and deleting branches are
  not possible at all.
- **Z.ai's new GLM-5.3 flagship is now a full cloud model option.** Same 1M
  context and thinking support as GLM-5.2, priced at $1.40 / $4.40 per million
  input/output tokens. This completes the three-model addition started with
  DeepSeek V4 — Gemini 3.7 Flash was already added in the previous release.

### Changed

- **The app starts faster.** The terminal engine — over 500 KB — no longer loads
  before the window is drawn, cutting a further third off the code loaded at
  start-up. It loads the first time you open a terminal.
- **Long conversations stay responsive.** Chat now draws only the messages on
  screen instead of every message in the conversation, and reconnecting to a
  running session no longer redraws once per word of the backlog.
- **Searching your project's index is faster** — roughly ten times, for repeated
  searches in a session.

## [1.0.0-beta.35.1] - 2026-08-15

### Fixed
- **The "llama.cpp is serving ... but this request needs managed-..." warning no
  longer fires when nothing is wrong.** Picking a local model stored an internal
  registry id that goes stale when the same file is re-registered, and the
  mismatch warning compared names instead of models — so it nagged on every
  message while the right model answered. Model identity is now resolved to the
  actual file on both sides before any warning, a real mismatch warns once
  rather than every message, and the warning names models the way the picker
  does instead of showing internal ids or file paths.
- **What's New shows this release's notes, not "[Unreleased]".** The dialog
  picked the top section of the changelog whatever it was; it now finds the
  section for the version you are running, and a release can no longer be
  tagged with an uncut changelog at all.

### Changed
- **The app loads noticeably faster.** The Settings and Help panels — over 40%
  of the interface code — no longer load before first paint. They load from
  disk the first time you open them, which at local speed is imperceptible.

## [1.0.0-beta.35] - 2026-08-15

> **If beta.34 showed you "lost backend connection", this update is the fix.**
> beta.34's installer shipped without the backend's runtime dependencies — the
> app window opened, but the engine behind it could not start, which surfaced
> as a lost backend connection on launch. Nothing was wrong with your machine
> or your data; the release packaging omitted a directory. Updating to this
> version replaces the broken install completely. The pipeline defect that
> allowed it has been fixed and is now verified on every release build three
> ways, including a start-up test of the actual packaged backend, so a build
> with this defect can no longer be published.
>
> **Use the installer, not the in-app updater.** A beta.34 install has no
> working backend, and the backend is what drives the update flow — so the app
> may not be able to update itself. Download the installer for your platform
> from the releases page and run it; it replaces the install in place.

> **You may be asked to re-approve browser site persistence.** If you had
> allowed the agent browser to stay signed in to sites, this version asks
> again, once, through the real consent dialog. That is deliberate — earlier
> builds granted persistence through a development flag rather than the
> consent flow, so consent is being re-collected properly. It is not a bug and
> your saved sessions are not lost; declining is what clears them.

### Added
- **Concentrate is available as a cloud provider.** It is a gateway: one API key
  reaches 172 models across 21 upstream providers, so you can try models from
  several vendors without opening an account with each. Add your key under
  Settings -> Cloud API Keys as with any other cloud provider — it is stored the
  same way, and Air-Gap mode blocks it the same way. The model picker lists the
  full catalogue grouped and ordered by upstream rather than as one flat list.
- **Lifecycle hooks now have a screen.** Settings has a Hooks panel where you
  can write your own hooks, approve or revoke the hooks a project ships in its
  own config, and — the part that was previously invisible — see the hooks that
  failed to load and why. A typo'd event name, a missing command, or a `hooks:`
  block left in the command-line-only config file used to be dropped with
  nothing but a line in the log; the panel now names each one. If you have ever
  written a hook and watched it do nothing, this is where you find out why.
- **A permission-profile editor.** You can create and edit the named permission
  profiles that decide what the agent may do without asking. Until now the
  profiles existed but there was no way to author them, which made the setting
  that selects one effectively inert.
- **You can pick which llama.cpp build Bodega installs.** Settings -> Models ->
  My Models -> llama.cpp engine now has a Build type dropdown listing the builds available for your
  machine (CUDA 12, CUDA 13, Vulkan, CPU, and so on), alongside the automatic
  pick it has always made for you. It also shows which build is actually on
  disk, so you can tell whether a choice has taken effect rather than assuming
  it. Previously the automatic pick was the only pick, and the setting that
  looked like it controlled this did nothing.
- **The agent can string a few browser steps together in one go.** Finding the
  right page used to cost a whole turn per action — navigate, look, click, look
  again — which is slow on a local model and burns through its patience before
  it reaches the thing you asked about. It can now send a short sequence of
  navigate / click / read-the-page / screenshot steps as a single request.
  Nothing about your approvals changes: each step still asks exactly what it
  would have asked on its own, at the moment it happens. Typing into a field and
  submitting a form are deliberately left out of these sequences — those always
  come to you as their own request, against the page as it stands right then. And
  if a step doesn't find what it was looking for, the rest is abandoned and the
  agent is told which step failed, rather than carrying on against a page it
  never actually reached.
- **The agent can now read a file by line number, not just by character offset.**
  Everything else it works with already speaks lines — search results, stack
  traces, type errors — so asking for "line 412" used to mean guessing which
  character that was. A read can now say which line to start at and how many
  lines to return, and the text comes back numbered like `cat -n`. Very long
  lines (a minified bundle, a one-line JSON blob) are shown cut short with a
  note saying so, instead of one line swallowing the whole window.
- **Jupyter notebooks are refused politely instead of dumped.** A `.ipynb` file
  is JSON with the code mixed into the saved output, and any plot in it is
  stored as an image blob. Reading one filled the conversation with data nothing
  could use. The agent now gets a short explanation of why it got no content.
- **Muse Glimmer 30B added to the model catalog.**
- **Qwen3.8-27B added to the model catalog.** Qwen's 27B dense vision-language
  model: 262K native context, native tool calling, thinking on by default, and
  it reads images through its own projector. Needs llama.cpp build b7990 or
  newer, which the catalog now checks before offering the download.
- **You can update the llama.cpp binary from Settings → Models → My Models → llama.cpp engine.**
  Until now the only install path was the one-time onboarding screen, so once
  you were set up there was no way to move off the build you first installed —
  which is a problem when a newer model needs a newer build. The panel shows the
  build you have and the build Bodega expects, and offers an update when they
  differ. Updating is always something you ask for: nothing updates at launch or
  in the background, and if llama-server is running you're asked to stop it
  first — a running executable is never overwritten.
- **The app now shows which build it's running.** Commit, timestamp, and
  whether it's a dev or packaged build now appear in Settings → About and in
  the diagnostics bundle.

- **Ask the agent about a GitHub issue or PR without leaving the conversation.**
  It reads the title, body, state, and for a pull request the diff and review
  comments, as structured data rather than a scraped page. Read-only — it never
  comments, merges, or changes anything. Large diffs are trimmed and say so, with
  the total, so nothing is quietly cut short.
- **Custom agents work in Chat Mode.** The same agents you've built for Code
  Mode, selectable from the chat header, off one shared list — an agent isn't a
  copy per mode. Chat Mode already ran the same tools underneath, so an agent
  behaves the same in either place.
- **The agent browser is available in Chat Mode**, in a drawer you open when you
  want it.
- **Take back a declined site.** If you decline "keep the agent signed in" for a
  site, that choice sticks across restarts — deliberately. Settings → Safety now
  lists declined sites with an **Ask again** action, so changing your mind no
  longer means editing a file by hand.

- **The agent browser has its own panel now, so you can watch a localhost preview
  and let the agent browse at the same time.** Until now those shared one space:
  the moment the agent opened a page, your Preview panel went blank, and you had
  to pick one. They're separate panels now — open the Agent Browser from the
  activity bar or with `/browser`, drag it wherever you like, and your dev server
  keeps rendering next to it.
- **Your chat now shows what it's about.** Bodega already gave every conversation
  a name based on your first message, but it only ever appeared in the sidebar
  list. The name now sits at the top of the chat itself, so when you come back to
  a window you left open you can tell at a glance which conversation you're in.
  Long names are shortened with the full text on hover, and a brand-new chat
  shows nothing until it has something to name itself after.
- **The agent can now fill in forms — search boxes, login fields, anything with
  a text input.** Until now the agent browser could look at a page and click
  things, but had no way to type into it, which made basic tasks like logging
  in or searching structurally impossible. It now can, with several layers of
  protection: a password (or other credential) field is always refused
  outright — there's no way to approve typing into one, and the agent is told
  to ask you to type it yourself. Any other value is scanned for things that
  look like a leaked API key or token before you're ever asked to approve it.
  Every single type asks for your approval, showing you the exact field and
  value, every time — nothing is remembered from one type to the next.
  Clicking a form's submit button after the agent has typed into that form
  now goes through the same real-values approval a direct submit uses,
  instead of the lighter "you're clicking a button" approval.

- **Persistent site logins for the agent browser — now live, if you turn them on.**
  `Settings → Safety → Keep the agent signed in` is enabled again, and turning
  it on asks you to confirm first. It stays off until you do: if you had it on
  in an earlier build, it has been reset, because that earlier switch approved
  a feature that was never actually wired and the confirmation you'd have seen
  didn't say what this one says.
  Grant a site and it gets its own cookie jar, kept apart from every other site
  and from the rest of the app. Revoke one site or all of them from Settings at
  any time, and turning on Air-Gap mode wipes every one of them, naming the
  sites before it clears them. Sites you decline are remembered, and there's now
  an **Ask again** action if you change your mind.
  This is new, and one part of it is worth saying plainly: a late fix corrected
  a bug where a login didn't survive a restart and browsing away from a granted
  site could write cookies into that site's jar. The fix is covered by tests but
  hasn't been through a full manual pass yet. If a login doesn't survive a
  restart, or you see one site's data show up under another, that's a bug worth
  reporting rather than expected behaviour.
- **A pending project skill now tells you it's waiting.** When you open a project
  whose own `.bodega/skills/` haven't been approved, a chip shows how many are
  waiting and links straight to Settings to review them. Previously the skill
  simply didn't load and the only trace was a line in the log. Approval is tied
  to the file's exact contents, so editing an approved skill asks again. Also
  available from the CLI as `bodega skills trust` and `bodega skills approve` —
  but **not yet against the engine the current CLI bundles.** The released CLI
  pins an older engine that has no `skills/trust/approve` route, so those two
  commands error until the next CLI release refreshes the engine. Approve from
  the app in the meantime.

- **Agent browsing beyond localhost — opt-in, off by default.** The agent
  could already drive your own dev server in the Preview tab; it can now,
  when you turn on `browser.widened_enabled` in Settings, navigate to real
  websites too — in an isolated, non-persistent browsing session with no
  shared cookies or storage with the rest of the app. Every new site needs
  your one-time approval. Two things always ask again separately, even on an
  already-approved site: submitting a form, and loading an address whose
  query string carries data you didn't see in the original approval — a
  mechanical guard against a page quietly appending your data to a link.
  Private/local-network addresses stay blocked no matter what. Never
  available under Air-Gap mode, and turning Air-Gap on mid-session
  immediately tears the browsing session down. This is opt-in with real
  residual risk — see the in-app Help page on agent tools for the honest
  version, not a "solved" one. Turn it on/off in Settings → Safety → Agent
  Web Browsing (non-localhost) — enabling asks for confirmation first, and
  the change takes effect on your next new chat or session, not mid-turn.
- **Ask Bodega about Bodega.** The agent knew almost nothing about the product
  it runs inside, and was told not to discuss it — so a question like "can this
  read PDFs?" got a shrug, or worse, a confident guess. It now ships the help
  documentation as searchable data and can look things up, answering about its
  own tools, modes and settings from what is actually documented rather than
  from imagination. It still will not repeat your project paths, your settings
  values or anything sensitive back to you.
- **Approval learning: you can now accept the suggestions it makes.** Approval
  decisions you make during agent sessions are saved locally to your database,
  and after you approve the same kind of moderate-risk shell command five
  times in Ask mode with no rejections, Bodega suggests learning it. Review
  and accept or dismiss suggestions in Settings → Privacy & Safety, and revoke
  any accepted rule at any time — revoking takes effect immediately, on the
  very next matching command. Accepting only changes Act mode: Ask mode always
  asks you first, no matter what you've accepted. Accepting a rule is
  app-only — a headless CLI run can never widen its own permissions.

- **Bodega can read PDFs, Word, Excel and PowerPoint files.** It used to decode
  any file you pointed it at as text — a PDF came back as a page of garbled
  characters, reported as a successful read, with nothing to tell the agent or
  you that it hadn't actually read anything. It now converts PDF, DOCX, XLSX
  and PPTX (plus RTF and legacy `.doc`/`.ppt`/`.xls`) to readable text, both
  when the agent opens a file itself and when you attach one to a chat message.
  Scanned (image-only) PDFs and encrypted files are correctly reported as
  unreadable rather than silently producing garbage. Other file types the agent
  can't decode — images, unknown binaries — are now named and sized instead of
  being fed through as mojibake.
- **The command-line tool can now propose and apply a small memory or persona
  change.** `bodega refine "<instruction>"` computes what it would change and
  shows you before doing anything; add `--apply` to actually write it, and the
  change can be undone with `bodega harness revert`. This had been built and
  tested in the command-line tool for a while with nothing on the other end
  to answer it. **The app half is now here — but the released CLI still bundles
  an older engine that lacks the `harness/refine` route**, so `bodega refine`
  errors until the next CLI release refreshes that pin. It works end to end
  against this app build; it does not work end to end from the shipped CLI yet.
- **You can now install plugins from inside the app.** A plugin bundles a set
  of skills and MCP servers into one folder or `.zip` (the same open format
  other AI tools use, so plugins built for elsewhere generally work here too).
  Settings → Integrations → Plugins lets you pick a folder or zip, shows
  exactly what it contains — every skill, every MCP server, any elevated
  permissions it's asking for — before anything is written, and nothing is
  approved automatically: you tick the specific permissions you want to grant,
  skill by skill. An MCP server that needs network access while air-gap mode
  is on is flagged right in the preview instead of failing silently. Installed
  plugins are listed with what they added, and can be removed — removal only
  touches what that plugin itself installed, never something you separately
  created under the same name. This was previously CLI-only.
- **"Manage allowed sites" in the agent browser's overflow menu**, in both Code
  Mode and Chat Mode. It opens Settings → Safety directly, so changing which
  sites the agent may stay signed in to no longer means hunting through
  settings from a different panel. It reads and writes the same grant list the
  consent card does — there is no second place where permissions live.
- **Settings now say which mode they apply to.** Every settings section is
  labeled Code Mode, Chat Mode, or both, and a new help page spells out what
  each one actually affects in each mode. Nothing moved and nothing changed
  behaviour — this is labeling, because "does this apply to my chat?" was not
  answerable from the UI before.

### Changed

- **The "Coming soon" strip is gone from Models -> Discover.** Every model listed
  there is now one you can actually download. The two small draft models that
  were sitting behind that label — Qwen 3 1.7B and Llama 3.2 1B, used to speed up
  a larger model — are downloadable, because the feature they were waiting on has
  shipped. The one entry that had no downloadable version at all was removed.

- **The beta period no longer expires.** Builds carried a hard cutoff of
  2026-11-01, after which activation returned an error and the app showed a
  terminal lock screen. That's gone — not pushed further out, removed. If you
  activated under any earlier cutoff, you come back active on your next launch;
  nothing to re-do.

- **Malformed tool calls are now counted.** When a model produces a tool call
  Bodega cannot parse, the recovery costs a full step of the task. That has
  always been true and never been measured. It is now recorded per model tier, so
  the decision about whether to enforce stricter output formats can be made from
  data instead of a hunch.

### Fixed
- **macOS builds sign again, and Linux installers drop a dead native binary.**
  The document converter added in this cycle ships one native build per
  platform; the packaging step was copying its symlinks in a way that pointed
  outside the app bundle, which macOS code signing rightly refuses, and Linux
  installers were carrying a second copy of the converter built for a C library
  they can never use. Packaging now keeps links relative, ships exactly one
  converter build per installer, and fails the release build loudly if either
  rule is ever broken again — including on the Intel-Mac build, which is
  produced on Apple-silicon machines and previously picked up the wrong build.
- **An approval card could show one command and approve a different one.** When
  two tool calls of the same kind were waiting for approval at once, both rows
  showed a card, and both cards resolved the same single pending approval — so
  the row reading `rm -rf build` could carry the Allow button belonging to the
  `ls -la` request. A card now binds to the exact request it is showing, and to
  the session it was raised in, so an approval in Chat Mode can no longer appear
  over a Code Mode conversation.
- **A conversational turn that tried to use a tool no longer answers with a
  blank bubble.** Some questions are answered without tools; if the model
  reached for one anyway, the attempt was discarded and you saw an empty reply,
  or worse, "I'll open that page now" with nothing behind it. Bodega now says
  plainly that the step was not carried out, and asking again runs it. Questions
  phrased as an instruction ("yes, and take a screenshot") are also routed to
  the tool-capable path in the first place.
- **A custom agent's own instructions reached the model on some turns and not
  others.** On a small context window Bodega builds a shorter system prompt, and
  that shorter version left out the custom agent's prompt entirely — along with
  the built-in Chat/Code instructions. Everything else about the agent still
  worked, so it looked active while behaving like the default one. It affected
  local models only, and only since the August context-window change, which is
  why the same agent behaved correctly on a cloud model and lost its personality
  on a local one in the same conversation. The shorter prompt now carries those
  instructions, trimming them if they are very long rather than dropping them,
  and says so in the log when it trims.
- **A custom agent that points at a model you do not have now says so.** If the
  model a custom agent is pinned to was never downloaded, selecting the agent
  quietly fell back to whatever was loaded. Settings -> Custom Agents and the
  agent picker both flag the missing model now.
- **When llama.cpp fails to start, you get the reason.** The server's own error
  output was being read and thrown away, and the automatic retry a couple of
  seconds later erased it — so a model that crashed on load looked the same as
  one that started fine. The error is now kept, written to the log, shown in the
  model status, and included in a diagnostics bundle. Stopping a model on
  purpose is still reported as a stop, not a crash.
- **Model status updates as soon as a model finishes loading**, instead of on
  the next refresh up to 30 seconds later. The periodic refresh is still there
  as a fallback.
- **The agent browser records both answers when it decides about a redirect.**
  It only ever wrote down refusals, so "allowed" and "the check never ran" left
  the same trace — no trace. The same was true of declining to open a browser
  under Air-Gap, which claimed in its own source to be recorded and was not.
- **The transcript now names the model that actually answered.** If a provider
  is set to serve one model at a time, or a local server already has a different
  model loaded, your request is quietly answered by whatever is resident. The
  assistant message is written the moment you press send — before any reply
  exists — so it could only ever be stamped with the model you *asked* for, and
  nothing corrected it afterwards. A message could therefore carry a model name
  that had nothing to do with it. The name is now rewritten from what the
  provider reports on the wire, and only when the two differ.
- **You are told when your model choice is overridden.** A single-active-model
  provider silently substituted its own default on every turn. A warning for
  exactly this case had been built and was never displayed anywhere. It is now a
  warning toast, once per response, naming both models and where to change them.
- **The browser no longer reports "navigated" for a page that never opened.**
  The agent's navigate said it had succeeded as soon as the request was handed
  off, not when a page actually appeared. A site that redirects — reddit.com to
  www.reddit.com, for instance — could have its redirect cancelled, leave the
  panel empty, and still be reported as loaded, at which point the agent would
  describe a page it had never seen. Success now means a document loaded. A load
  that dies reports the failure and tells the agent not to claim the page is
  open; a load still in flight is reported as neither.
- **Eight local models were hidden from Discover on 32 GB machines.** Catalog
  entries carry a minimum-RAM figure, and a 32 GB machine does not report 32 GB
  — firmware reserves some, so the operating system sees about 31. Every entry
  needing 32 GB therefore failed its own check and was filtered out of the list,
  including Qwen3.8 27B and a model some machines were running at the time. The
  minimum is now checked with a small tolerance, so a machine is not excluded by
  the number printed on its own box. Genuinely too-large models are still hidden.
- **Vision models no longer hand out a context window they cannot back.** A
  vision model loads two things onto the card: the weights and a separate image
  projector. Bodega only counted the weights when it decided how big a default
  context to ask for, so every vision model in the catalog sized its cache
  against between half a gigabyte and two gigabytes of memory that was already
  spoken for. The projector is now measured on disk and subtracted first. On a
  24 GB card with a 1.4 GB projector this is about 6,000 tokens of window that
  were previously promised and not there. **Still open:** the "fits on your
  machine" badge in Discover, and the VRAM timeline, are both still weights-only
  — a vision model can show as fitting and then start with a squeezed context.
  Widening that badge changes which models every machine is offered, so it is
  deliberately not in this release.
- **DeepSeek's reasoning model was billed at roughly six times its real rate.**
  `deepseek-reasoner` is served as the cheaper V4-Flash model, but the spend
  tracker still priced it against the retired R1 rate. It now shares Flash's
  price entry, and Bodega notices in general when a provider serves a different
  model than the one asked for instead of quietly pricing the one it asked for.
- **Scheduled and cached prices are now understood.** Rates that change on a
  date, and peak/off-peak windows, used to be flattened into a single number
  that went wrong the day the schedule changed; cached input tokens were billed
  at the full input rate. Both are now priced against the clock the usage was
  stamped with. **Still open:** Anthropic charges 1.25x for writing to the cache
  and there is no rate for that yet, so a cache-cold Anthropic turn still reads
  slightly cheaper than it was.
- **Switching llama.cpp build types actually reinstalls now.** Once you were on
  the current build, asking for a different build type quietly did nothing and
  reported success — the download was skipped because only the version number
  was being compared.
- **Two GPUs are no longer read as one.** llama.cpp spreads a model across
  every card of the same brand, and Bodega already added up their total memory,
  but the "how much is free right now" reading still came from a single card —
  so a two-card machine could be told a model wouldn't fit when it would. Free
  memory is now measured across the same set of cards, and only when every one
  of them reports a figure; if any card is silent, Bodega falls back to the
  single-card reading rather than publishing a number that is quietly short by
  a whole GPU.
- **A malformed read position is refused instead of silently reading the top of
  the file.** Asking to read from `"2abc"` or from position 1.5 used to be
  quietly turned into position 0, so the agent got the beginning of the file
  back and had no way to know it had asked for something else — and would ask
  again the same way.

- **Saving a memory, project memory, or project instructions could fail
  outright** on a database that had already taken a recent update. Fixed.
- **Unified-memory machines (DGX Spark, Strix Halo, and similar) were told
  they had no GPU.** Hardware detection expected a separate GPU and CPU
  memory pool; on a machine where they're the same pool it reported zero
  GPU layers, labeled the machine "No GPU detected — Minimal tier," marked
  every catalog model as not fitting, and recommended the smallest model
  available. It now recognizes this class of hardware. Apple Silicon was
  already handled, but its usable-memory estimate assumed all system RAM
  was available to the model — Metal actually grants roughly 70-75% of it,
  so the estimate is now closer to what's real.
- **Multi-GPU: llama.cpp now gets credit for pooled VRAM across matching
  cards.** It splits a model across same-vendor GPUs by default, but
  Bodega's fit and recommendation checks only ever counted the largest
  single card, so a model that genuinely fit across two cards was marked as
  too big. Ollama still gets only the largest single card, because that's
  how it actually loads a model. Cards from different vendors are never
  pooled together.
- **A model that needs a newer llama.cpp than you have is now blocked
  before download**, with a message naming the build it needs and the one
  you're running. Muse Glimmer 30B needs a specific build; loading it on an
  older one crashed the local model server.
- **MLX no longer appears as a provider on Windows and Linux** (it's
  Apple-Silicon only), and MLX auto-detection can no longer mistake a
  running llama.cpp server for an MLX one — they were sharing the same
  port.
- **A relative file path read with no project attached now says which
  folder it actually resolved against**, instead of silently reading from
  wherever the path happened to land.
- **Agent-browser screenshots and tool calls used to disappear from the
  transcript the moment a turn finished**, so you couldn't expand and
  review them afterward. They now persist with the message and stay
  readable after the fact, in both Chat and Code mode.

- **The agent-browser toggle and custom-agent picker were missing from Chat Mode's empty state.** They
  only appeared once a chat was already in progress, since the header that holds them wasn't mounted on
  the greeting screen. It's mounted now, from the first screen.
- **Sending the agent to a real page could crash the whole browser panel.** If a `navigate` landed before
  Electron had finished attaching the webview, it threw synchronously and took the panel down with it.
  That now resolves as a retryable tool error instead of crashing.
- **Chat Mode sometimes showed raw `tool_call {...}` markup, with a stray `/think` token, as the
  assistant's answer.** The parser only recognized one wrapper format; a response in a different but
  still tool-shaped format now goes through the normal retry path instead of being shown as-is.
- **The agent reported a stranger's account as yours.** Asked who was signed in, it read the author of
  the first post in the feed and answered with that handle — confidently, and wrong every time the top
  post wasn't yours. It now says it can't tell, because the page doesn't actually reveal it. An honest
  "I don't know" replaces a confident guess.
- **The agent-browser panel's width in Chat Mode reset to default on every reload.** Your chosen split is
  now remembered.
- **A project skill waiting on approval now tells you.** Open a project whose own
  `.bodega/skills/` haven't been approved and a chip appears with the count,
  linking straight to Settings. Before, the skill simply didn't load and the only
  trace was a line in a log. Approval is tied to the file's exact contents, so
  editing an approved skill asks again.
- **Search suggested an option that doesn't exist.** When a code search found
  nothing it advised trying a case-insensitive search — `code_search` has no such
  setting, so the retry was identical and found nothing again.
- **Queued messages that failed stayed in the queue forever**, shown as though
  they were still going to run.
- **The agent browser's own findings surface in code mode**, not only in chat.

- **Agent browser screenshots weren't showing up in Code mode.** The screenshot
  thumbnail depended on a session pointer that Code mode's agent panel never set,
  so nothing rendered even though the capture itself worked correctly. It now
  reads the right session and the screenshot shows up in the stream.

- **Closing the agent browser and then asking the agent to open a site again now
  actually shows it.** If you closed the agent browser's own X button and then
  asked the agent to revisit the same site, the agent would navigate there
  successfully — but the panel stayed hidden until you clicked "Open agent
  browser" yourself, even though the page had really loaded. Any agent-driven
  visit now brings the panel back into view, whether it's a new site or the
  same one as before. The X still closes it, and it stays closed until the
  agent does something new.
- **Approval cards for the agent browser now say more about what's actually
  happening.** The "allow agent browsing" prompt used to show only the site's
  domain — now it also shows the specific page the agent is asking to open,
  when that's more than just the homepage. The "click" prompt used to show
  the whole page address as if it were the site name; it now leads with the
  site plainly, so you don't have to parse a URL to see where you're
  approving a click.
- **Quitting Bodega One now frees the VRAM your local model was using.** Bodega
  runs your local model (llama.cpp) as its own background process so it can
  hand it off between chats without reloading it every time. But on Windows,
  quitting the app force-closed everything so fast that the local model
  process never got the chance to shut itself down — it kept running in the
  background, still holding onto several gigabytes of GPU memory, until you
  noticed it in Task Manager and killed it by hand. Bodega now tells the model
  to unload cleanly the moment you quit, so that memory is back and available
  for anything else within a couple of seconds. An earlier attempt at this fix
  told the app the model had shut down without actually checking — it now
  waits and confirms the model process is really gone before saying so, and
  tries again if the first attempt didn't take.
- **The agent's browser window could cover the composer and model picker,
  making them unusable.** When the agent browsed a website, the browser
  window would sometimes land right on top of your message box and model
  picker — so typing a follow-up could go into the website instead of to
  Bodega, and clicking the model picker could click the website instead. The
  browser window now avoids that area automatically, and it's also
  draggable now if you want to move it out of the way yourself.
- **A model you downloaded while Bodega was open didn't show up in the model
  picker until you restarted the app.** If you ran `ollama pull` (or added a
  model any other way) while Bodega was running, the model picker kept
  showing the old list — even though Bodega's own connection check could see
  the new model just fine. Restarting was the only way to make it appear.
  There's now a refresh button right in the model picker's search box that
  looks again for real, so a newly downloaded model shows up without
  restarting anything.
- **Vision worked for downloaded llama.cpp models but never for ones you added yourself.**
  If you picked a vision model from the built-in catalog, Bodega automatically
  grabbed its matching "vision helper" file for you. But if you already had a
  vision model on your computer and pointed Bodega at it directly (sideload),
  it had no way to find that helper file — vision silently never worked, even
  though everything looked registered correctly. Bodega now looks in the same
  folder as the model for a plausible helper file and pairs them automatically
  when there's exactly one reasonable match. If it's not sure (for example, two
  different vision models sharing one folder, or a helper file that doesn't
  look like it belongs to your model), it leaves them unpaired instead of
  guessing wrong — pairing the wrong files can crash the model server. If you
  already hit this, you don't need to do anything: just open your models list
  again and it fixes itself automatically. (There's no separate "re-scan"
  button in the app yet — opening the models list is what triggers the check.)

- **Long converted documents were quietly cut off at 50,000 characters, with
  the agent told it had the whole file.** A page reader's pagination is meant
  to keep going; this had capped every read at a fixed limit and reported it
  as complete. It now paginates like everything else, so the agent can keep
  reading a long document instead of reasoning from a partial one.
- **Built-in skills were missing from every installed copy of Bodega.** The
  twelve skills that ship with the app were documented, listed in the help, and
  never actually installed — the app looked for them in a folder that was not
  included in the build. Slash commands that relied on them did nothing. They are
  now included, and a packaged build that finds none of them now says so loudly
  at startup rather than starting up looking healthy. The command-line tool needs
  the same fix in its own release.
- **Undoing a remembered note or a saved knowledge card now actually works.**
  The command-line tool could ask to undo one of these two kinds of changes,
  but the app had no way to do it and always refused. Both are undoable now,
  restricted to your own account, and a request that can't be safely undone
  still refuses rather than pretending it worked.
- **Memory searches quietly ignored a filter that was never populated.** Memory
  rows carried a branch field that nothing ever wrote, while three search paths
  filtered on it. Nothing was broken in practice, but the code implied a feature
  that did not exist. Removed rather than half-built.

- **Local models that think before answering were being told not to.** Qwen3
  and similar models reason by default. Bodega was sending an explicit
  "don't think" instruction on every request unless you had opened Settings
  and changed it — so anyone who never touched that setting got the weaker
  path. It now stays out of the way unless you actually turn thinking off,
  and turning it off still works. Bodega had already measured what this costs
  on a different model family and fixed it there; this closes the same hole
  on the local path.
- **GLM models had no thinking toggle at all.** The control never appeared,
  so there was no way to turn their reasoning off. It appears now.
- **The agent can check its own work again.** After writing a file, an agent
  that read it back to verify what it had just done would have that read
  cancelled and be told to stop reading and write more files. Reading a file
  you just wrote is now allowed, once per file per write, and no longer
  counts against you as time-wasting. The guard that stops an agent exploring
  forever without writing anything is unchanged.

- **A turn that read more than eight files got none of them.** When an agent
  read a lot of files at once, a message reminding it to be selective was
  inserted in a position that caused every one of those file reads to be
  discarded before the agent saw them. It received the reminder and nothing
  else. Present since 1.0.0-beta.31.10.5. Six more places with the same
  ordering mistake were found and fixed, and there is now a check that reports
  this class of failure instead of losing the content silently.
- **Verification could grade a write against the wrong file.** A written file
  was matched to what the task asked for by filename alone, so a write to
  `test/index.ts` could be checked against a deliverable declared as
  `src/index.ts` — passing or failing the wrong file. Matching now respects
  the directory when one was specified, with a bare-filename fallback for
  deliverables that didn't specify one.
- **Bodega's own answer to "what does this app do?" pointed at the wrong
  tool for reading PDFs, and named tools that weren't documented at all**,
  including the tool that answers questions about Bodega itself. Six missing
  tools were documented, and the retrieval that decides which help page to
  show got two rounds of fixes: a question phrased differently than its
  matching heading could return nothing, and a page that just happened to
  repeat a search term many times could outrank the page that actually
  answered it.
- **The first-run hardware check could hang indefinitely on "Detecting...".**
  A timeout existed but only covered some of the places that triggered the
  check; a stuck GPU query in one of the others left new users stuck on the
  loading screen with no way forward. The timeout is now built into the
  check itself, so every caller is covered.
- **An abnormally-closed backend could leave the local model server running
  and holding onto VRAM.** Bodega now checks for and cleans up its own
  orphaned model server at startup.
- **A background chat session sharing the model server with the main window
  was budgeted against double its real context window**, because the shared
  budget wasn't being divided by how many sessions were actually sharing it.

- **A note or fact you explicitly asked Bodega to remember never lost
  confidence over time, even after it stopped being true.** Memory entries
  built from things you said in passing correctly fade in relevance the
  older they get. Entries you saved on purpose (`save_memory`, or anything a
  tool wrote directly) were being treated as permanently at maximum
  confidence instead — they never faded and never got flagged as possibly
  stale. They now decay too, more slowly than an inferred fact since you
  said it on purpose, but not forever.

- **Reading part of a file could authorise overwriting all of it.** The guard
  that stops the agent writing a file it has not read only checked whether the
  file had been read at all, not how much of it. Read 8% of a large file, write
  the whole thing back, and the other 92% was gone with nothing to warn you.
  The guard now tracks which ranges were actually read and refuses a full-file
  write on a partial view. The related case — where the agent had matched an
  exact snippet rather than read the file — used to be refused with a message
  that made no sense ("you have read only 0 of 0 characters"); it now says what
  it actually wants.
- **The read tool advised a line range it could not yet honour.** On a large
  file it told the model to read a line range at a point when the tool still
  only addressed characters. The advice was accepted, silently dropped, and the
  same read repeated from the start — a loop the tool caused itself. The tool
  now genuinely supports line-addressed reads (see "read a file by line number"
  above), so the advice and the behaviour agree.
- **Reading a Windows file and writing it back no longer rewrites every line
  ending.** Line endings were being converted to Unix style on the way in, so
  content the agent showed you and then wrote back came out as a whole-file
  diff. The file is now left as it is on disk; a byte-order mark is still
  stripped, since it broke snippet matching.

### Security

- **What you approve is now what runs.** When the agent asked permission for a
  tool call, a later step could still adjust that call's arguments after you had
  already said yes — so an approval prompt showing one value could be followed
  by a slightly different value actually executing. Three such adjustments
  (a near-miss operation name being corrected, a file-tool alias being
  normalised, and a missing argument being filled in) now all run *before* the
  prompt is built, on both approval paths, so the prompt shows the final call.
  Nothing here needed re-approving, because there is no longer a difference to
  re-approve.
- **"Strict" command sandbox is now actually stricter than "Moderate".** The two
  settings ran the same check, so choosing Strict changed nothing you could
  observe while Settings and the docs described it as tighter. Strict now means
  no shell command auto-approves — every one waits for you, read-only included.
  The Settings text and the docs also said the sandbox applies in Ask mode; it
  is the opposite. Shell commands never auto-approve in Ask or Plan mode, and
  the sandbox level governs Act mode only. Both now say so.
- **Closed a gap in where a saved cloud API key may be sent.** The check that
  stops a stored key from being sent to an internal address was missing carrier
  NAT (100.64.x), the 0.x range, the broadcast address, and integer-form IPs —
  ranges the app's other network guards already blocked. `web_fetch` likewise
  refused the broadcast address only after a DNS lookup, which it skips for a
  literal IP. All the private-range guards are now held to one shared list of
  adversarial addresses by a test, so a range added to one and forgotten in
  another fails loudly instead of silently.

- **A project Air-Gap Vault now blocks choosing a cloud provider, not just using
  one.** The check that refuses to select a cloud model while air-gap is on was
  written to consider the project's vault, but every caller in the shipped app
  left that argument empty, so it only ever read the global switch. A project
  with its own Air-Gap Vault on, while the global toggle was off, could still
  set a cloud provider as its primary — the largest outbound path in the
  product. All six call sites now resolve the active project's vault. Nothing
  here says anyone's data was taken; it says the vault was narrower than the
  setting implied on this path, and now matches it.
- **A project Air-Gap Vault now also holds outside the tool layer.** Turning on
  the vault for a project always stopped that project's tools from reaching the
  network, but several things that are not tools were still checking only the
  global air-gap switch: the verification and review layer (which embeds file
  contents and diffs in what it sends), the mixture fan-out, HTTP hooks, wiki
  and repo-map generation, and skill learning. With the global switch off and a
  project vaulted, those could still call out. They now read the project's
  vault. Nothing here says anyone's data was taken — it says the guarantee was
  narrower than the setting implied, and now matches it. A source-level check
  was added first so the full list was enumerated rather than guessed at, and
  it fails the build if a new one appears.
- **Embeddings and inline code completion now honour the vault too.** An
  earlier draft of this entry said they didn't — that was wrong, and the fix
  shipped in this same release. Inline completion (fill-in-the-middle) reads
  the project's vault before it will call out, and the embedding service
  applies the vault both to cloud embedding backends and to any non-local
  embedding address. If you rely on a project vault rather than the global
  switch, these two are covered.
- **One embedding path is still global-only, named so you know the edge.** A
  single lower-level embedding call site checks the global air-gap switch
  rather than the project's vault. In practice it sits behind the service-level
  gates above, but if you want a hard guarantee for that path today, use the
  global switch rather than a per-project vault. This is the one remaining gap
  and it is on the list to close.
- **The memory embedding write and a memory-row uniqueness check are now
  scoped per user**, closing two places that assumed a single account
  rather than checking whose data they were touching. Bodega is a
  single-user product today, so this is hardening ahead of the case where
  it isn't — not a fix for a leak anyone hit.

- **A project with air-gap on could still reach the network through two tools.**
  `github_context` (which carries your GitHub token) and `consult_mixture` (which
  can send code to cloud models) each checked only the *global* air-gap switch, so
  a project-level air-gap with the global switch off didn't stop them. Both are
  now blocked at the same gate as every other outbound tool. Reaching GitHub also
  can no longer become a standing auto-approval.
- **"Wipe all sites" now stops pages before clearing them.** A page still running
  on a site you were signed into could write its cookies back moments after the
  wipe. Revoking a single site already worked this way; wiping everything didn't.
- **Approving a project skill validates the name**, so a crafted request can't
  reach a file outside the project's skills folder.
- **The agent's browser could end up painting somewhere the panel isn't.** Opening
  the Preview tab moved the browser panel, but the page kept drawing at its old
  spot — usually off the bottom of the window, so the browser looked empty or
  vanished. It tracks the panel properly now, including when you drag the panel
  to a different part of the layout, and the page doesn't reload when you move it.
- **Agent browser screenshots taken right after navigating could come back tiny
  and useless (a sliver of pixels instead of the page).** The agent would
  screenshot a page it had just opened before the browser view had actually
  finished laying itself out, so the capture caught the in-between moment
  instead of the real page. It now waits for the view to be properly sized
  before capturing, and if a capture still comes back too small it retries
  once automatically. If it's still bad after that, the agent is told
  plainly that the screenshot failed (with the size it actually got) instead
  of being handed a useless image and left to guess.

- **A safety check on the agent's browser had never actually run.** A backstop
  meant to stop an ungated page from loading in the agent browser was reading a
  field that doesn't exist in the version of Chromium we ship, so it quietly did
  nothing every time instead of doing its job. It now uses a signal that's really
  there, and blocks the page from opening at all rather than just cancelling the
  first load. Nothing was exposed by this — the checks in front of it were doing
  the work — but a backstop that never fires isn't a backstop.
- **Bodega now cleans up leftover browser data it has no record of.** If cookie
  storage for a site was left on disk without a matching "keep me signed in"
  record — something that could happen during testing — it was invisible in
  Settings and couldn't be removed from there. Bodega now finds and clears that
  data at startup, so what Settings shows you is what's actually on disk.
- **Closed a gap where a specific kind of private key could slip past the output
  filter unredacted.** Bodega scans command output and previewed page content for
  things that look like leaked secrets before showing them to the AI. That scan
  recognized several private-key formats, but missed the plain, un-labeled kind
  used by many modern tools — it could have shown up unredacted. It's now caught,
  along with a few other secret formats (OpenRouter, Google, and Azure keys, and
  "Bearer" tokens) that weren't recognized before.
- **The bug-report export (repro bundle) had its own, older secret scanner that
  missed some of the same formats.** If you generated a bug-report bundle to
  share with someone, an OpenRouter key, a Google API key, or a short GitHub
  fine-grained token could have slipped through unredacted in that file — even
  though the same secret would already have been caught elsewhere in the app.
  The bundle exporter now uses the same secret-detection list as the rest of
  Bodega, so it can't drift out of sync again.

- **A shell command could smuggle a second command past approval.** Bodega
  judges how risky a shell command is by splitting it into its parts and
  looking at each one. It split on `|`, `||`, `&&` and `;` — but not on a
  single `&`, which is what separates commands on Windows, nor on a line
  break, which separates them on macOS and Linux. So `git status & something
  else` was read as one harmless command, scored as safe, and in Act mode ran
  without asking you. Both separators are now recognised. Affects the app and
  the command-line tool, which share the same check.
- **A project you cloned could load skills nobody approved, with their full
  permissions intact.** Skills you import yourself already get a review step
  and have their risky abilities stripped out until you approve them. A skill
  that shipped inside a project's own `.bodega/skills/` folder skipped that
  step entirely — it loaded on clone with every permission it asked for, and
  could even be picked automatically just from its description matching what
  you asked for. Project skills now go through the same approval gate as
  imported ones, keyed to the exact contents of the skill file so an edit to
  it re-asks. If the approval record can't be read, nothing is trusted.

---

## [1.0.0-beta.34] - 2026-08-05

### Added
- **Connect to hosted tool servers directly.** Bodega could only run tool servers
  as a local program on your machine. Hosted ones - GitHub, Linear, Notion,
  Sentry - had to be wrapped in a local shim first, if you knew to do that. You
  can now add one by its address, with a token, from Settings. Air-gap mode
  refuses them and says so up front rather than letting you fill in a form that
  cannot work.
- **Bodega Observatory.** The Map tab now shows what has been verified, not just
  what exists. Each file carries its latest verification result, a rail lists
  recent findings and files that have grown past their size limit, and every row
  opens the file at the right line. Off by default; turn it on in Settings under
  AI Behavior. Files nothing has verified show a neutral mark rather than a
  green one - absence of a check is not a pass.
- **Prompt caching now covers the conversation, not just the preamble.** On
  Anthropic models, Bodega already reused the cached system prompt and tool
  definitions between turns. The conversation itself was re-sent and re-billed in
  full on every step of a long task. It is now cached too, and the cached region
  rolls forward as the conversation grows. How much this saves depends on how
  often the conversation gets summarised, since summarising rewrites the history
  and starts the cache over.
- **A record of what each request is actually made of.** Every request now
  reports how its context was spent - the fixed preamble, the project state, the
  per-turn additions, the tool definitions, and the conversation - alongside how
  much was read from cache. Visible in the debug panel. This exists because
  almost every claim about context size in this project was an estimate read off
  the code rather than a measurement.
- **Per-file verification results for edits, not just new files.** Verification
  produced no per-file result at all when the agent modified existing files,
  which is most real work. It does now.
- **Two new Qwen cloud models.** Qwen 3.8 Max, Alibaba's new flagship, and Qwen
  3.7 Flash, a cheaper vision-capable tier, are both available now through the
  Qwen (Alibaba DashScope) provider. Qwen 3.7 Flash is also available through
  OpenRouter.

### Changed
- **In code mode, Bodega now writes a rough version of the file you asked for
  early, then improves it in place**, rather than working the problem out in
  the terminal and only writing a file once it has the full answer. Debugging
  and questions that do not name a file to produce are unaffected.
- **Turning thinking on or off now works for cloud Qwen models.** The control
  had no effect at all when talking to Qwen through Alibaba's own API - the
  model kept reasoning (or not) on its own regardless of what you picked. The
  toggle now reaches the model.
- **The newest Qwen models were losing their reasoning by default.** Fixing the
  toggle above introduced a second problem: once the toggle reached the model,
  it reached it every time, including when you never touched it. Two of the
  Qwen models this release adds show their reasoning by default on Alibaba's
  own API; leaving the control alone was silently turning that off. It now
  takes an explicit choice from you to turn thinking off on those models -
  leaving the control alone keeps their own default.
- **The paid licence activation screen has been removed.** Nobody had ever
  purchased through it, so no existing activation is affected. Beta email
  activation, which is how every current user signs in, is unchanged.
- **The installer is about 40% smaller** - 909 MB down to 553 MB unpacked, and
  the Windows installer is now 136 MB. Most of that was ours rather than the
  framework's: unused copies of libraries that were already bundled, every
  language pack for an English-only app, development-only packages shipped to
  users, and - the largest single piece - stale development builds that were
  never cleared before packaging. Earlier releases very likely shipped those
  development builds.
- **Files the agent edited show a neutral mark rather than a green tick.** A
  green tick used to appear on an edited file purely because the file existed
  afterwards, which is true of every edit whether it worked or not. Existing is
  no longer treated as evidence of anything on an edit, so edited files show a
  neutral mark unless something real was checked - a test, a proof gate, a
  framework check. Newly created files are unaffected. You will see fewer green
  ticks on ordinary work; nothing has broken.

### Security
- **A leftover benchmarking setting could no longer widen where the agent is
  allowed to write.** A setting used only by the internal benchmarking
  harness to let a disposable test container write outside its task folder
  was read from an environment variable, and environment variables are
  inherited by child processes. An earlier attempt at this fix cleared the
  variable in the app's own startup process, but the backend can also be
  launched on its own - by the command-line tool, for example - and that path
  was never covered. If the variable happened to still be set in the terminal
  a launch started from, an unattended run (an overnight batch, a background
  session, or a headless run started from the command line) could write
  anywhere on disk instead of being kept inside the project, with nothing
  shown to the user. The setting is no longer read from the environment at
  all. It now has to be passed explicitly to the backend process itself, a
  channel nothing inherits automatically, so nothing a parent shell happens
  to have set can reach it. Any time the write boundary is actually widened
  is still logged.
- **Sending data out through a one-line script is blocked properly now.** The
  agent can run short scripts inline, and the rule meant to stop those from
  reaching the network was checking for the wrong thing. Measured against real
  commands it had it backwards: it blocked ordinary local work like running a
  build, while letting four common ways of posting a file to a remote address
  through untouched. The check now looks at where the script is actually sending
  data. Reaching a remote address is refused; reaching a server on your own
  machine still works, so local development is unaffected.
- **A command written in your prompt is no longer run automatically during
  verification in an interactive session.** This release taught Bodega's
  verification step to also run the exact command you name in your own
  request - "verify by running the tests," for example - so a wrong claim of
  success can be caught against your own stated check, not just Bodega's
  guesses. That command now only runs on its own in an unattended run (an
  overnight batch, a background session, or a headless run started from the
  command line), where you have already accepted that the agent executes
  shell without asking each time. In an ordinary chat session it no longer
  runs by itself; every other part of verification - compiling, running your
  project's own test suite, checking the file exists and looks right - is
  unaffected and keeps running exactly as before, in every mode.

### Fixed
- **The thinking control was missing for models installed through the app's
  own model manager.** Models pulled or sideloaded via the app's llama.cpp
  model manager get an internal id that the composer's Thinking toggle could
  not read a model family from, so the control silently never appeared, even
  for models that support thinking. The id is resolved to the real model name
  before that check runs now.
- **Local models on a consumer GPU ran out of room before they started work.**
  Bodega sizes a local model's context to what your VRAM can safely hold. On a
  large model that lands in a middle band - big enough not to count as a small
  window, small enough to be tight - and the full fixed preamble was being sent
  anyway, taking roughly half the available space before the conversation
  began. Long tasks then spent their budget compacting instead of working.
  Local runs in that band now get the short preamble. Cloud models are
  unaffected.
- **The model picker was showing every model a provider offers, including ones
  that cannot hold a conversation.** A cloud provider with a large catalog -
  Qwen/DashScope alone can list over 150 - dumped everything into one flat,
  alphabetical list: text models mixed in with text-to-speech, image
  generation, video, translation, and embedding models, plus half a dozen
  dated copies of the same model. Finding the current flagship meant scrolling
  past all of it or typing its version number from memory. The picker now
  shows chat-capable models first, with the newest version on top, collapses
  dated snapshots down to one row, and keeps everything else one search away
  rather than gone - nothing is ever permanently hidden.
- **A long task that runs out of time mid-answer is now reported as having run
  out of time, instead of presenting a cut-off answer as complete.** Bodega
  could stop a response partway through when its own time budget ran out, but
  still show the partial text as if the model had finished normally.
- **A task could be reported as passing while its own test command was failing.**
  When a task states how to check itself, for example a test command or a build
  step, Bodega runs it as part of verification. That result was scored as one
  signal among several, so a high score elsewhere could outvote it and the work
  was reported complete with the stated check still red. A failing check now
  vetoes the pass outright, as does one that never ran. Verification is stricter
  as a result, so some work that used to be reported as done will now come back
  for another pass.
- **Long tasks spent up to a fifth of their time asleep, on purpose, for nothing.**
  Bodega paces its own requests so it does not trip a provider's per-minute
  token limit. When a single request was larger than that whole per-minute
  budget, the pacing code searched for a point in the next minute where the
  request would fit, found none, and then waited the maximum 45 seconds anyway
  before sending it unchanged. No amount of waiting can make one oversized
  request fit inside a smaller budget, so the wait bought nothing. A long coding
  conversation crosses that line routinely, which meant a 45-second pause on
  most later steps. Measured across 217 recorded benchmark runs, this was 18.8%
  of total elapsed time. Bodega now sends the request immediately in that case
  and relies on the provider's own response and the existing retry, which is
  what happened after the wait regardless.

  Two related problems went with it. The default per-minute budget used for
  providers that do not publish one was 40,000 tokens, well under a normal
  agentic conversation, and is now 100,000. And a personal endpoint on your own
  machine or network was being paced against a cloud limit it does not have,
  which is now treated as local and not paced at all.

  One trade-off worth stating: if your plan has a genuinely tight token limit
  and your provider sends no rate-limit information with its responses, Bodega
  now discovers that limit by hitting it rather than by anticipating it. Set
  your real limits under `llm.rate_limit_overrides` if this affects you.

  This closes the specific waste - waiting a fixed amount of time for nothing -
  but it does not make pacing go away. A long task that generates a lot of
  output in each step will still cross a provider's per-minute limit and still
  wait for it; that wait is legitimate now, and how much of it you see mostly
  tracks how verbose the model's answers are, not this fix.
- **Claude Opus 5 Fast could not be used at all.** It was listed as a model, but
  no such model exists: every message sent with it selected failed immediately
  with a not-found error. Anthropic offers the faster tier as an option on the
  standard Opus 5 model rather than as a separate model, and Bodega was not
  requesting it. Selecting it now sends what the faster tier actually requires.
  Note that access to it depends on your own account having the capacity
  available; without it, Anthropic returns a rate-limit error rather than a
  reply.
- **Turning thinking off now turns thinking off.** On local models the control
  had no effect at all - the setting was discarded before it reached the model,
  which kept thinking regardless. On the newest Claude models, Off and Fast Mode
  relied on simply not asking for thinking, which those models treat as thinking
  on. Both now say so explicitly. On Claude models served through another
  provider, where the control could never have worked, it no longer appears.
- **Cloud Boost with Gemini as the boost model now respects your reasoning
  setting.** It was silently dropped - Boost sent every request through a path
  that only knew how to turn reasoning on or off for OpenAI-style models, and
  Gemini's own reasoning parameter is shaped differently, so the choice never
  reached the model. Boost requests to Gemini now carry it.
- **Blocked commands now say which rule stopped them.** The agent was told only
  that a command was blocked, so it would retry variations of the same thing
  instead of trying a different approach. It now names the kind of rule and,
  where there is one, suggests a way to do the same job safely.
- **Searches that find nothing now explain why.** An empty result looked the
  same whether the pattern was wrong, the path did not exist, or the search
  could not handle a pattern spanning multiple lines. That last case now works
  instead of silently finding nothing.
- **DeepSeek was running with its reasoning turned off.** DeepSeek reasons by
  default. Bodega's reasoning control defaults to "off", and for every other
  provider that means "send no instructions and let the model behave normally".
  For DeepSeek it meant something different - an explicit instruction not to
  think - so anyone who had never opened that setting was using the model with
  its headline capability disabled.

  Measured on a public benchmark of 89 programming tasks: 33 solved with the
  instruction being sent, 47 without it. Same model, same tasks.

  Only the default changed. Choosing "Off" yourself still turns thinking off,
  and a per-model or per-message choice still wins as before.
- **The task-list reminder reaches the model again.** It was skipped whenever the
  turn's text began with a brace, which is most turns while tools are running, so
  on long tasks the agent gradually lost track of its own plan.
- **The agent stopped refusing its own ordinary commands.** Three safety rules
  were matching far more than they were written to match, and the agent could
  lose whole tasks retrying variations of a command that was never going to be
  allowed.

  Deleting a build directory - `rm -rf build-output` inside your project - was
  read as an attempt to wipe the entire filesystem, because the rule accepted
  any absolute path rather than the root itself. The rule now looks at how deep
  the path goes: the top of the filesystem is refused, a directory inside your
  project is not.

  Anything containing the letters of a shutdown command was refused, including
  `-halt-on-error` (an ordinary LaTeX flag), `-no-reboot` (an ordinary emulator
  flag), and searching your own logs for the word. These are now recognised by
  position - a shutdown is a command being run, not a word appearing in a flag
  or a search string.

  Reading a private key was confused with any variable or property named `key`,
  so `print(item.key)` in a one-line script was refused as if it were leaking a
  certificate. The check now distinguishes a file path from a property name.
  Reading an actual key file is still refused, and every one of these rules
  keeps blocking the dangerous form it was written for.
- **Verification no longer marks a check failed when it could not run.** A
  correct, working file could be scored 35 out of 100 and reported as a failure
  with nothing actually wrong. Checks that do not apply to what you asked for -
  no framework to look for, no content requirements to match, no test to run -
  were counted as zero rather than left out, so the score was measured against
  work that was never part of the request. A one-line Python script that ran
  correctly could not reach a passing score at any setting. Those checks are now
  excluded from the total, and the report says "n/a" for them instead of showing
  a zero out of thirty that reads like a failed test. Files are also credited for
  existing even when they are short - a small file is not a missing one.

  Two related repairs: a task with no framework to check was quietly awarded
  those points for free, which is the same mistake in the opposite direction; and
  a check that was attempted but could not be judged - a missing interpreter, a
  timeout - no longer counts against the score, because it observed nothing.

  This is a real improvement but not a complete one. The one-line script above
  went from 35 to 50 and still does not pass. Closing the remaining gap means
  removing a minimum-length rule on file content, and doing that let a stub Go
  file certify as verified on a check that inspected nothing. That is worse than
  the problem being fixed, so the rule stays until the verifier can tell the two
  cases apart.
- **The iteration limit you set is now the one that is used.** If you raised the
  limit for how long the agent may keep working, that choice could be quietly
  overruled and a longer task would stop early. The app was deciding whether you
  had chosen a value by checking whether a row existed in its settings, which is
  not the same question - routine internal updates write rows too, and a value
  that happened to match the built-in default was indistinguishable from never
  having been set. Your choice is now recorded when you make it. If you set the
  limit before this release, save it once more so it is recorded properly.
- **Add Server said nothing when it could not be used.** Adding a tool server
  with the name left blank did nothing at all and gave no reason. The button now
  says what is missing.
- **Compacting a short conversation no longer contradicts itself.** Pressing
  Compact announced that it was compacting, then immediately said there was too
  little to compact. It now reports only what happened.
- **Long answers from DeepSeek were being cut off far too early.** The two newest
  DeepSeek models were configured to stop after roughly a twenty-fourth of the
  output they can actually produce, and that figure was sent to the service as a
  hard ceiling rather than just recorded on our side, so long generations really
  were truncated. Their two general-purpose models were also carrying the context
  size of the previous generation. Corrected against the live service rather than
  the documentation.
- **Your answer was being saved twice.** Every reply was written to the
  transcript two times, by two different parts of the app that did not know about
  each other. One copy carried the token count, the other carried the model's
  reasoning, and neither had both. Now one copy is written, with everything on it.
  This also means the conversation replayed back to the model no longer contains
  each of its own answers twice. Overnight and scheduled runs get the model's
  reasoning saved for the first time; they never had it. One case is knowingly
  left: a reply interrupted part-way through is still written twice.
- **Asking for help no longer depends on which language you named.** When a
  request was too vague to act on, Bodega would ask a couple of questions before
  starting. Whether it asked at all depended on the language: a vague Python
  request got questions, an equally vague Bash, Node, PowerShell or shell request
  got none. Now they all do. Well-specified requests are still left alone.
- **Saying "C++" or "C#" no longer gets you asked what language you want.**
  Neither name could ever be recognised, so a request naming them was treated as
  naming nothing.
- **Suggestions when Bodega asks which framework you want.** It could only offer
  options for Python, TypeScript and Ruby, and stayed silent rather than ask for
  anything else. It now covers JavaScript, Go, Java, Rust, PHP, Swift, Kotlin,
  C, C++ and C#.
- **Long tasks on a small context window stop losing their place every few
  steps.** Part of the context window is taken up by fixed material that cannot
  be summarised away. Bodega was measuring how full the window was against the
  whole window, including that fixed part, so on a small window it summarised the
  conversation roughly every third step and reclaimed almost nothing each time.
  One task spent eighty-six tool calls re-reading the same nine files. It now
  measures against the part it can actually reclaim, and will not summarise when
  there is nothing worth reclaiming.
- **Summaries stopped piling up on top of each other.** Each summarisation left
  the previous summary in place and added another, so the space available shrank
  a little more every time and the next summarisation came sooner. Summaries now
  replace rather than accumulate.
- **A background task could have its working copy deleted while it was still
  running.** When the agent delegates a piece of work, it does that work in a
  separate copy of your project. The cleanup that removes abandoned copies could
  not tell a live one from a crashed one, and ran every fifteen seconds. It could
  remove the copy out from under the running task, and separately could offer it
  to you as an abandoned copy to confirm deleting. Both are fixed, and the final
  delete step refuses a copy that is in use regardless of how it was asked.
- **"Nothing to apply" is no longer reported when the check itself failed.**
  Comparing a background task's work against your project could fail silently and
  report zero changes, which is indistinguishable from a task that genuinely
  changed nothing. It now says the comparison failed.
- **Tools from connected servers now ask permission in chat, not only in code
  mode.** With approvals set to ask, tools provided by a connected external
  server ran without prompting in chat mode. They prompt everywhere now.
- **Connected servers with more tools than fit on one page.** Only the first page
  of a server's tools was read, so the rest silently did not exist.
- **Server arguments containing spaces.** An argument with a space in it - a
  Windows path, a piece of JSON - was split into pieces before being passed on.
- **Tool results that are not text.** A result containing only an image was
  reported as a success with nothing in it, and an error carrying no text was
  reported as a success outright. Both now say what actually happened.
- **Cancelling a tool now cancels it on the server.** Stopping a run, or a tool
  hitting its time limit, left the request running on the other end with nobody
  waiting for it. The time limit is also configurable now.
- **Recorded checks from overnight runs.** Verification results from headless and
  scheduled runs were never recorded, so nothing from those runs was ever
  re-checked later for regressions.
- **A file with almost nothing to check no longer reports a confident pass.** A
  plain text or data file, where you did not say what it should contain, was
  graded on two things - that it exists and is not empty - and then reported
  100 out of 100. The score is unchanged; what it says is not. It now reads as
  unverified rather than passed, which is what it always was.
- **Quoting something in passing no longer invents a requirement.** Bodega looked
  for phrases like "containing" or "with the text" anywhere in your message and
  then treated every quoted string as required file content. Mentioning a quoted
  name in an unrelated sentence could fail work that was correct. The phrase now
  has to actually be describing the file it precedes.
- **When a forbidden library costs points, Bodega says which one.** It quietly
  deducted and moved on, so a result could come back lower with no explanation.
- **Asking for something "without using X" is now understood.** It was ignored
  entirely - only libraries Bodega inferred were incompatible counted, never the
  ones you ruled out yourself. A rule you stated is now enforced; one that was
  merely inferred still only deducts, because a guess should not block work.
- **File ranges expand based on what is on disk.** Asking for "file1 to file5"
  now checks whether those files already exist before deciding you meant a range
  of new ones, instead of trying to read your intent from the sentence.
- **A pass with nothing behind it is treated as unverified, not as a refusal.**
  A separate check added earlier this release asks whether a pass is backed by an
  actual test or run, not just the absence of a failure - because a whole test
  suite silently failing to run reads the same as it not existing. That check had
  a bug: languages whose contracts only ever run a compile check - TypeScript,
  Go, Rust, Java, Ruby, PHP, and configuration or documentation tasks - never
  produce anything the check counts as a test in the first place, so it was
  refusing all of them regardless of whether the work was correct. It now tells
  "nothing to check" apart from "checked and came back empty," and only the
  second one counts against the work. Before this fix, that distinction did not
  exist for those languages: every contract of theirs read as "checked and came
  back empty," whether the work was right or not. You may still see overnight
  Loops park instead of applying, and GitHub automation open a draft pull request
  instead of a ready one, but now only when a test genuinely ran and came back
  empty - not merely because the language does not have one to run.

---


---

## [1.0.0-beta.33] - 2026-07-27

The verification release. Bodega used to tell you your finished work had failed —
measured against an independent grader across 89 real tasks, it solved 35 and
reported failure on 29 of them. That is fixed, along with the reverse case, and
about forty other things.

### Added
- **Claude Opus 5.** Available as a cloud model, along with a faster variant.
  1M context, and the app picks the right thinking behaviour for it
  automatically.
- **A setting for how long a single run may take.** If you run the agent inside
  something that stops long jobs on its own — a CI job, a test harness — you can
  now give the agent a shorter budget than the thing running it, so it finishes
  and reports properly instead of being cut off mid-task.
- **A preferred provider list for OpenRouter.** OpenRouter passes your request
  on to one of several companies that actually run the model, and it does not
  keep you on the same one between requests. Each of them remembers your recent
  context separately, so bouncing between them meant that memory was almost
  never there to reuse and every request started over. You can now name which
  ones you prefer, in order. On a long task this cuts both the bill and the
  waiting, and if a preferred one is busy or down your request still goes
  through elsewhere rather than failing. Off by default, and it changes nothing
  for any other provider.

### Security
- **A secret sitting next to ordinary text is redacted again.** Shell output is
  scanned for credentials before you or the model ever see it. The scanner
  measured how random a run of characters looked, but it measured the whole run
  it happened to grab — so padding a key with sixty characters of filler
  averaged the randomness away and the key printed in full. It now measures a
  sliding window, so filler next to a secret no longer hides it. It also strips
  terminal colour codes before looking: a single escape sequence in the middle
  of a token used to break every pattern we match.
- **A detached side chat now honours the project's privacy vault.** The vault is
  a per-project switch that keeps that project's work off the network. Which
  project applied was decided by the app window making the request, and a side
  chat popped out into its own window does not know which project is open — so
  it sent nothing, and the vault quietly did not apply. The project is now
  determined by the session itself, and a request cannot override it.
- **Project scans stay inside the project.** A shortcut or junction inside a
  folder pointing somewhere else on the machine was followed, and file paths and
  code symbols from outside the project could end up in the model's context.

### Changed
- **Kimi K3 is selectable.** Moonshot's current flagship — one million tokens of
  context, image input, reasoning always on — was already supported but was not
  named anywhere in the setup, so there was no way to know the model id. The
  Kimi provider now lists it.
- **The verification and drift strips above the composer look like the rest of
  the app.** The quality-check card painted its whole background red, amber or
  green, which no other status surface in Bodega does; both strips now use the
  same neutral panel with a small status dot, and sit together as one block
  instead of two competing bars. The drift row also dropped its jargon — it now
  says "Check whether previously verified results still hold" rather than naming
  an internal mechanism.

### Fixed
- **Long tasks stop forgetting what they already made.** On a task with more
  steps than fit in the model's context, Bodega summarises the earlier part of
  the conversation to make room. That summary is written by the model, and on a
  smaller local model it routinely dropped the record of which files had already
  been created - so the agent made the same file again, hit the loop guard that
  exists to stop exactly that, and stopped early with only part of the work done.
  A five-file task finished with two. Bodega now restates, as plain fact after
  each summarisation, which files were asked for and which already exist on
  disk. Read from the disk itself, not from a record of what the agent believed
  it had done.
- **Asking for "3-5 files" is understood as a range.** When you gave a count as a
  range, only part of it was read, so the finished-work check could be measuring
  against the wrong number of files. Ranges are now read as ranges. And when the
  agent cannot work out what you asked for, it says the request is incomplete and
  asks, instead of picking a number and building against its own guess.
- **A range of files you asked it to READ is no longer mistaken for files to
  create.** "Merge part1.csv to part9.csv" was read as a request for nine new
  files, because the word "to" was treated as a range marker regardless of what
  the sentence asked for. The finished-work check then reported failures for
  files that were never supposed to exist, and the agent went off trying to
  produce them. We tried several times to work out from the sentence whether a
  range meant "read these" or "create these", and it could not be made reliable,
  so the app no longer guesses. A range written with "to" or a hyphen is taken as
  the two files you actually named, and the task is reported as unverified rather
  than reported as a success over a partial list. A range written with "through"
  still expands into every file in it. Numbering that changes width partway
  through a range, like part8 through part12, is counted correctly now.
- **On Windows, a command that throws away its output no longer leaves a file
  called `$null` in your project.** Commands run through the Windows command
  prompt, where `$null` is an ordinary filename rather than a way to discard
  output, so a command written to keep quiet quietly created a file instead. It
  succeeded, so nothing reported anything. The output is now discarded properly,
  and the agent is told about the correction so it stops writing it. Text inside
  quotes is never rewritten, so a commit message that happens to mention `$null`
  is left exactly as written, including messages containing escaped quotes,
  which the first two attempts at this got wrong. Where it cannot be certain
  whether the text is quoted, the command is left untouched.
- **The Skills switches work.** Turning a skill off, or turning off automatic
  skill activation, silently failed every time and reverted with an error. Both
  switches sent the setting in the wrong shape, so it was rejected before it was
  ever stored.
- **Editing a skill no longer erases it.** The editor loaded the skill's text
  from an address that did not exist, failed quietly, and saving then wrote the
  empty result over your instructions. Saving is now refused when the text
  could not be loaded.
- **Three ways to lose unsaved work are closed.** Closing a tab from its
  right-click menu discarded unsaved edits without asking, unlike every other
  way of closing a tab. Quitting with unsaved files discarded them with no
  prompt. And opening certain files — a binary, or anything very large — from
  the breadcrumb bar, the problems list, or a terminal link produced an empty
  editor whose first save truncated the real file on disk to nothing.
- **A dropped connection no longer runs your task twice.** If the connection
  broke mid-answer, the app resent the request while the original was still
  running, so file edits and commands could be applied twice. It now asks the
  backend whether the original is still alive and rejoins it. If it cannot
  reach the backend to ask, it stops and tells you so rather than guessing —
  reopen the session to see where the run got to. Resending on a guess is how
  the work got done twice in the first place.
- **That check now covers the case it was written for: the connection dropping
  while the answer is still arriving.** It only handled a connection that failed
  to open, or one that closed cleanly, so the most common break of all went
  straight past it. Losing the network partway through a reply now does the same
  thing as any other drop: the app asks the server whether the run is still going
  and rejoins it. Before, it either resent the turn, which could apply the same
  file edits twice, or sat there looking like a slow model with nothing to tell
  you.
- **Losing the connection in the first seconds of a reply is handled too.** When
  the app asked the backend whether your run was still alive, the backend only
  counted a run as started once the model produced its first output. Everything
  before that answered "nothing is running" — and with a local model that gap
  covers loading the model into memory and can last tens of seconds. A
  connection lost in that window got a wrong answer and the request was sent
  again, so the same task could run twice. The backend now claims the session
  the moment it takes the request, not when the model first speaks, and it holds
  that claim even if you close the window, because a run that carries on in the
  background is still a run. Losing the connection while a local model is
  loading no longer risks the request being carried out twice.
- **Reloading the window asks before discarding unsaved files**, the same as
  quitting. Ctrl+R used to throw the work away without a word.
- **Quitting from Chat Mode asks too.** The unsaved-work check only existed
  while the editor was on screen, so quitting from chat discarded open edits
  silently — and made every quit from chat wait two seconds first.
- **Preview actions do not repeat themselves after a reconnect.** Reconnecting
  replays the run to catch you up, and anything the agent had clicked in the
  preview was clicked a second time.
- **OpenAI's Europe endpoint is recognised as OpenAI.** It was being treated as
  an unknown local server, which meant a short connection timeout, no model
  list caching, and no usage or cost reporting.
- **"Persist memory" is gone, because it never did anything.** Nothing read it,
  and its description claimed memories were lost when the app closed. They
  were not.
- **Editing a project-scoped knowledge card keeps it in that project** instead
  of quietly making it global.
- **The app starts even when the terminal component cannot load.** A missing
  terminal library stopped the whole app from starting, with an unhelpful error.
  Now everything else runs and only the terminal reports the problem.
- **Local models are no longer cut off after five minutes.** A hard time limit
  killed a healthy answer mid-sentence. Long answers are now judged on whether
  they are still making progress.
- **Streaming no longer breaks on some providers.** A usage-reporting field was
  being sent to every recognised provider, including ones that reject unknown
  fields outright. It now goes only to providers known to accept it.
- **Answer-quality results no longer show a wrong score card.** A checked answer
  displayed five red zero-scores under a green pass. Those bars do not apply to
  answers and are gone; the actual finding is shown instead.
- **Side chat fixes.** Switching sessions showed one conversation's messages
  under another's heading, and the "send this to the main chat" action could
  fire into the wrong session, or one that had been deleted.
- **A failed run notifies you even if you walked away** — previously the
  notification was skipped in exactly that case, leaving the session stuck on
  "running".
- **Finished work is no longer reported as failed.** This is the big one. The
  quality check treated "I could not observe this working" the same as "I watched
  this fail" — so a task the agent had genuinely completed came back marked
  failed, the command exited with an error, automated runs refused to apply the
  change, and the agent was sent off to repair work that was already correct. On
  a run of 89 real-world tasks measured against an independent grader, we solved
  35 and told you we had failed on 29 of them. Two rules caused nearly all of it:
  a task was failed when nothing had exercised the new behaviour, and a task was
  failed when a piece of text we expected to find in a file was not there. Now
  only something we actually watched go wrong can fail a task. When we cannot
  confirm the work, it says so plainly instead of calling it broken, and it does
  not go back and redo work you have already paid for. A pass still means the
  same thing it always did: we saw it work.
- **A missing test no longer counts as a failed one.** The reverse case, found in
  the same review. A check that could not run — because the environment was not
  there, or because it timed out — was allowed to sit alongside a check that
  really did fail, and the real failure could be outvoted. A check that observed
  a failure now always carries.
- **Long runs stopped ending as errors.** When the agent reached the time limit
  you set, one path finished cleanly and reported its work, while another threw
  the run away as if something had gone wrong. Reaching a limit you configured is
  a normal ending, and it now reports as one, with whatever was completed.
- **The agent gets the number of steps you asked for.** Setting a higher step
  limit had no effect: an internal recommendation quietly overruled it, often to
  a quarter of what was requested, and nothing said so. What you set now wins.
- **Writing a large file could be cut off partway.** A safeguard meant to catch a
  stalled model was watching for text coming back, but a model writing a long
  file sends it as instructions rather than text, so a productive run looked
  frozen and was stopped mid-write. Affects every provider except Anthropic,
  which was already handled.
- **Edits that would break a file are caught before they land.** A syntax check
  already guarded whole-file writes but not the targeted edits the agent makes
  most often. It now covers both, and only ever rejects an edit that breaks a
  file which was fine beforehand, so it cannot block a valid change.

### Changed
- **The agent stops second-guessing capable models.** A set of safety rails
  built for small local models were being applied to every model, on every task.
  They capped how many times a file could be edited, turned some edits into
  reads, and interrupted with reminders. On a capable model doing real work
  these cost steps and sometimes ended the run early. They now step aside for
  strong models working unattended, and stay exactly as they were for smaller
  models and for anything you're supervising.
- **Quality checks now act on what they find.** When a check catches a real,
  demonstrable failure — code that doesn't compile, a command that errors — the
  agent gets the actual error text back and fixes it, instead of the failure
  being noted and the answer shipped anyway.
- **Work done through the terminal now counts.** The agent often writes files by
  running shell commands. Progress tracking and quality checks only watched the
  file editor, so that work was invisible to them and many finished tasks were
  never checked at all.
- **The agent can do several things at once again.** Five separate places in its
  instructions told it to make one tool call per message. Nothing in the app
  required that, and it meant reading three files took three round trips instead
  of one. It can now group independent work into a single step, while anything
  that depends on an earlier result still waits for it. Small local models keep
  the one-at-a-time guidance, which is where it was always meant to apply.
- **Less filler in what the agent reads back.** Every result it got included
  things it could not use: the command it had just sent, echoed back verbatim;
  empty fields; timestamps it never looks at. That was roughly a tenth of
  everything it read, and on a long task it crowds out the parts that matter.
  Failures still carry the full detail, because that is when the extra context
  is worth its space.
- **Long tasks get the time they are actually allowed.** When the agent runs
  inside something with its own time limit, it now reads that limit and sizes
  its own work to fit, instead of applying one fixed budget everywhere. Tasks
  that were being stopped with most of their time unused now run to completion.
- **Project information is arranged for reuse.** The unchanging part of what the
  agent knows about your project — your rules, the file list, the repository map
  — now comes before the parts that change every turn. Providers that cache
  repeated context can reuse far more of it, which is faster and cheaper,
  and it helps local models most.

### Fixed
- **Requests could freeze for up to a quarter of an hour.** If a cloud provider
  stalled, each retry started its own fresh timer instead of counting against
  one deadline, so a single request could sit there for around fifteen minutes
  with no way to interrupt it. Affected every cloud provider, including Cloud
  Boost, where it also burned budget. There is now one deadline covering the
  whole request, and a provider asking us to wait can no longer ask for an
  unbounded wait.
- **The agent gave up early on longer tasks.** Some models were being given a
  much smaller step budget than they could actually use, because of how their
  size was detected, and turning the limit up in settings had no effect. Models
  now carry their own working length.
- **The agent refused to create project files.** Standard project manifests —
  `pyproject.toml`, `setup.py`, `Cargo.toml`, `go.mod` and similar — were
  treated as out of scope and blocked, repeatedly, on tasks whose whole point
  was packaging. They are now always allowed, and a refusal no longer repeats.
- **The agent stopped reading files that were right there.** On tasks phrased as
  "create" or "write", it could decide the folder was empty without checking,
  and refuse to read existing files. It now checks first.
- **Task lists didn't work outside the app.** Running the agent from the command
  line without a session, its to-do tracking silently failed on every run.
- **Internal notes could appear as the answer.** When the agent stopped itself
  because it was going in circles, the internal explanation was sometimes
  delivered as the reply. You now get a real answer, or an honest description of
  what went wrong.
- **Failed runs said nothing about why.** A run that ended without a verdict
  reported no reason at all. It now reports what happened, how long it ran, and
  how much it did.
- **Prompts starting with a dash were rejected.** The command line treated the
  first line of such a prompt as an unknown option and refused to start.
- **Edits could be reported as saved without being saved.** A safety check meant
  to make the agent read a file before overwriting it was performing the read
  instead of the write, and then reporting success. The agent believed its change
  had landed, hit the same failure again, and only discovered the truth by
  re-reading the file. A change whose target text is already in the file now
  simply applies, since finding that text proves the agent knew what was there.
- **Edits that removed code were silently dropped.** A second check discarded any
  rewrite that made a file smaller and reported it as done. Deleting the part
  that does not work is often the most valuable edit there is, and file size says
  nothing about intent. Rewrites now apply whenever the content actually differs.
- **Python files could not be saved on some machines.** If a machine has `python3`
  but no plain `python`, the syntax check treated the missing program as a syntax
  error in your code and blocked every Python file the agent tried to write. It
  now looks for both, and a missing checker means the file is saved, not blocked.
  The same fault made a missing test toolchain read as "your code fails its
  tests".
- **Quality checks could pass work that was never checked.** A task that edited
  existing files could score full marks on nothing more than the file having
  changed, with no check actually run. Overnight and automated runs act on that
  verdict, so unverified work could be committed as verified. A verdict now
  requires something observed, and a run that could not be graded says so instead
  of reporting success.
- **Quality checks read what the agent claimed, not what was on disk.** If a file
  was rewritten after its first write, or changed by a shell command, the check
  looked at the earlier version. Repairs were being verified against the problem
  they had just fixed. Checks now read the file.
- **The agent was refused ports nothing was using.** Two ports were treated as
  reserved whether or not anything of ours was listening on them, including when
  you are using a cloud model and the local model server cannot be running at
  all. The agent was told the port was taken, and spent its time hunting a
  process that did not exist. Ports are now only held when something is actually
  holding them, and if one genuinely is, the message says so plainly instead of
  sending you looking.
- **Ordinary data was being hidden as if it were a password.** The check that
  keeps secrets out of the model's view was matching on shape rather than
  content, so long runs of ordinary text — DNA sequences, hashes, encoded data —
  were blanked out. On one task the agent could not see the data it had been
  asked to work on. Secrets are still caught, including every recognisable key
  format, but ordinary data comes through.
- **Failures reported nothing useful.** A run that ended badly could report only
  "Something went wrong", which was also the sole record of it. The real cause
  now comes through. Related: provider error text was being sent on without being
  cleaned first, which could have carried an API key from a rejected request.
- **Runs could hang without producing anything.** Work done before the agent
  starts — reading your project, preparing context — had no time limit, so a slow
  step could consume an entire run in silence. Those steps are now bounded and
  skipped if they take too long, since they are optimisations rather than
  requirements, and a run that produces nothing at all now ends with a reason
  instead of waiting to be killed.
- **Long responses were being cut off.** A limit meant to stop stuck connections
  was also ending healthy ones, because it measured elapsed time rather than
  whether anything was still arriving. It now watches for actual silence, so a
  model that is thinking is left alone and a dead connection is still caught.
- **The agent stopped with its own plan unfinished.** It would write out the steps
  it intended to take, complete the first, and finish. It now keeps going while
  its own list has open items, within limits, and a run left with most of its
  time unused will not finish early.
- **Cost and token usage were missing for OpenRouter.** We were not asking the
  provider to report usage, so spend showed as nothing at all and looked like the
  provider simply did not supply it. Cost, token counts and cache usage now come
  through.
  **Worth knowing if you set a spending limit:** because that spend was not being
  recorded, limits on your own API keys had nothing to count and never stopped
  anything. They now work as written. The money was always being spent — it just
  was not being counted — so if you set a limit some time ago and forgot about it,
  this is the release where it starts taking effect.
- **The app could sit there doing nothing at the start of a run.** The backend
  reported itself healthy the moment it could answer at all, before its database
  had finished opening. A request sent in that window waited with no output and
  no error — in the worst case for twelve minutes — and because nothing had been
  sent yet, none of the usual stall detection could see it. The health check now
  distinguishes "running" from "ready", a request will not wait more than
  45 seconds before saying so, and a slow start is written to the log instead of
  being invisible.
- **A slow provider at startup could hold up everything.** Checking the model
  provider on launch had no time limit, so a provider that accepted the
  connection and never replied stopped the app from starting at all. It now gives
  up after 15 seconds and starts anyway, which is what already happened when a
  provider was simply offline.
- **A single command could be cut off at one minute.** Shell commands were
  allowed to ask for up to two minutes, but were stopped at one regardless. Long
  installs and builds died halfway, and — worse than losing the command — the
  agent could no longer run what it had just written, so it lost the ability to
  find its own mistakes.
- **The count of changed files was usually zero.** It was matched against a list
  of tool names that did not include the one the agent actually uses to write
  files, so ordinary edits were never counted. Reports of what a run changed were
  wrong, and anything reading that number treated finished work as if nothing had
  happened.
- **Tokens produced were reported far too low.** Only prose was counted, and
  everything the agent wrote through a tool — which is most of the code it
  produces — was invisible. Billing was never affected, but the usage figures
  were, and they were most wrong on exactly the runs that did the most work.
- **Runs that used up their time were never quality-checked.** Verification only
  ran when the agent decided it was finished. A run stopped by its time limit
  skipped the check entirely, so the work most likely to be incomplete was the
  work least likely to be examined.
- **A failed check could be talked past.** After one repair attempt, a failing
  proof — code that does not run — could no longer force another, so the run
  ended on the agent's explanation of the failure with most of its time still
  unused. Repairs are now allowed as long as there is time for them, and the
  check result decides, not the explanation.
- **"The file was written" was taken from the agent's word, not the disk.** Three
  separate places decided a deliverable existed by looking at what the agent had
  tried to do rather than what was actually there. A write to a similarly named
  file, or one that never landed, counted — and counting it switched off the
  safeguard meant to catch precisely that.
- **A stalled model could sit there for ten minutes and nothing noticed.** The
  agent watches for a stalled connection two ways, and both were watching the
  wrong thing. One reset its timer whenever any data arrived — including the
  "still working" pings some providers send, which are not progress. The other
  switched itself off permanently the moment the first word of a reply arrived.
  So a reply that started and then stopped, while the provider kept pinging, was
  invisible to both. Measured across a long test run, this quietly consumed more
  than three hours of working time. Both now watch for actual progress, and when
  one does stop a run it says what it saw instead of a bare timeout.
- **Quality checks did not run at all on nearly half of automated runs.** There
  are several ways an agent run can end, and only one of them was checking the
  work. A run that hit its time limit, or ended on an error, or stopped early
  after writing files, finished with no verdict at all — so the report said
  nothing, and anything downstream deciding whether to apply the work had nothing
  to go on. Every ending now produces a verdict or an explicit "could not check
  this, and here is why". Recovered verdicts never claim success: work that could
  not be properly checked is marked unverified and held for review.
- **A quality check could pass work whose own tests had failed.** Two separate
  ways: on edits, the score was a percentage of the checks that happened to run,
  so a thinner set of checks produced a *higher* score — "the file was touched and
  it still compiles" came out as full marks. On new code, individual checks were
  scored and then capped at the end, so two passing checks absorbed the penalty
  for two failing ones, and the report listed failures as blocking while the
  verdict passed anyway. Both are closed: a check that failed because the code is
  broken now blocks; a check that could not run because something was missing from
  the machine still doesn't count against you, but no longer earns a pass either —
  the run is held for review instead.
- **The agent copied your files instead of using them.** Given a task pointing at
  files outside the project folder, it would read them, recreate them inside the
  project, and work on the copies — and where it could not copy something, it
  wrote code to invent a replacement and used that. The cause was one line in its
  instructions claiming it could not touch anything outside the project, which was
  not true, and left it no other way to finish the job. It now works on the files
  a task actually names, and will tell you when something is genuinely off limits
  rather than working around it.
- **Naming a file you wanted read could get it overwritten.** When working out
  what a request asks for, every filename mentioned was treated as something to
  create. Ask it to read one file and write another, and both went on the
  create list — so the input could be overwritten, and the quality check then
  graded the file it had just been told to clobber.

---

## [1.0.0-beta.32.1] - 2026-07-24

### Added
- **Automatic model routing, with nothing hidden.** The agent can now pick the
  right model for each step of a task on its own: a fast model for reading and
  searching, a code model for edits, a stronger model for planning and
  verification. It always shows which model handled what, and if a step fails
  the quality checks it retries once on a stronger model and tells you it did.
  Off unless you turn it on for an existing install; on by default for new ones.
- **Point at a model on another machine.** You can now run against a model served
  on another computer on your network. Air-gap mode still refuses any non-local
  endpoint, so a remote model is treated as an explicitly online mode.
- **New Qwen models.** Added Qwen3-Coder-30B (2507) and a vision-capable local
  Qwen3.6-35B, plus the Qwen3.7-Plus cloud model.

### Changed
- **Plan reviews are readable now.** The "Review plan" card renders the plan
  as formatted text with step and file counts in the header, instead of a raw
  monospace dump.
- **Streaming responses render more smoothly.** Long answers no longer re-parse
  the whole message on every frame; completed sections are frozen while only the
  newest text updates, new words fade in as they arrive, and code stays readable
  while it streams. Tuned to feel good on local models.
- **The app is lighter on your machine.** Removed a settings write that happened
  every few seconds, shares one GPU query across the status strip instead of
  several, and narrows a set of components that were re-rendering on unrelated
  settings changes.

### Fixed
- **Some community models on Ollama returned nothing at all.** Models whose
  chat template insists on a single leading system message rejected every
  request, because the app splits its system prompt in two for cache
  efficiency. The app now detects that rejection, merges the system prompt,
  retries automatically, and remembers the model needs it permanently — no
  setting to flip, nothing to redo after a restart. Verified against the
  exact model from the report. Thanks to the user whose diagnostics bundle
  made this a ten-minute diagnosis.
- **No more silent stalls when the assistant wants to ask you something.**
  A clarification prompt could sit in dead air with nothing on the wire; the
  connection now stays visibly alive while the question waits. The assistant
  also no longer asks "what programming language?" when your request already
  says — a fully specified "write a SQL query" runs immediately.
- **Local models no longer lose the plot on a simple greeting.** Saying hi in
  code mode was being treated as a work order: the app forced tool execution
  and loaded the message with task scaffolding, which made mid-size local
  models invent a phantom prior task. Greetings are now recognized as
  conversation, the extra scaffolding stays out of them, and the coaching
  meant for genuinely weak models no longer fires for capable mid-size ones.
- **Task lists no longer fail when the model writes them the natural way.**
  The TODO tool rejected a properly formatted list and only accepted a quoted
  workaround, which could stall a run in a retry loop until it hit the
  iteration limit. Both forms are accepted now, for every tool.
- **A run that keeps talking instead of acting now delivers its answer.**
  After a few nudges, the app returns what the model wrote instead of burning
  the whole iteration budget and ending with "reached the iteration limit."
- **The GPU memory strip actually shows up now.** It was wired into a
  component the app never renders, so it was invisible in every mode. It now
  lives in the always-on status bar in both chat and code mode. On Windows it
  also reads free memory directly from the NVIDIA driver tools when the
  system query does not report it, and a machine whose GPU stats are
  unreadable shows a muted indicator instead of nothing.
- **Local models that write tool calls in their own invented syntax are
  understood anyway.** Mid-size models produce a different homemade wrapper
  around tool calls nearly every run; the app now recovers the call from any
  of the shapes seen in live testing, and when a shape is truly unreadable it
  shows the model the exact expected format and lets it retry instead of
  printing the raw markup as the answer.
- **Questions about what the assistant remembers now actually check memory.**
  "What do you know about me" and similar questions were routed down a fast
  path with no tool access, so the model either guessed or emitted a memory
  lookup nothing executed. Those questions now run where the memory tool
  works.
- **The Smart Auto toggle now lives in one place.** The old copy in the
  experimental tab (which could silently fight with the new Routing section
  over the same setting, including on unrelated saves) is gone.
- **The sidechat "inject into main" dialog is readable and clickable.** It now
  renders above the whole app with a blurred backdrop instead of inside the
  sidebar where panels bled through it, and a small notice above the composer
  shows that a block is queued for your next message.
- **A finished response can no longer be misread as cut off.** The last piece
  of a streamed response arriving without a trailing newline was dropped,
  which could trigger a needless retry or a false "truncated" notice.
- **Model routing display fixes:** the final step of a run (including a
  quality-check escalation) now appears in the routing strip, concurrent runs
  no longer mix their steps together, and switching sessions clears the
  previous session's context-fill readout immediately.
- **Assorted editor fixes:** a diff-review disposal race that logged errors
  during streaming edits, an undo that could close the review while the file
  still held the change (it now reports the failure instead), and several
  per-token re-subscriptions that wasted work during streaming.
- **The GPU memory strip no longer disappears on a slow or briefly failed
  reading.** On some machines the live GPU strip in the status bar could vanish
  entirely instead of showing its data. It now stays visible: it shows a brief
  "GPU" indicator while the first reading loads, keeps showing the last good
  reading through a momentary hiccup, and only hides when the machine genuinely
  has no readable GPU.
- **A stored cloud key can no longer be sent to an unexpected address.** When
  testing a provider connection without re-entering the key, the app now checks
  that the destination is the provider's own host before attaching your saved
  key, closing a path where a crafted address could receive it. Testing a local
  provider with a freshly typed key is unaffected.
- **Consistent wording and controls in the side chat.** The "add to main
  conversation" action now uses one consistent label, and the per-project
  offline switch now matches every other switch in settings.

## [1.0.0-beta.32] - 2026-07-17

### Added
- **Code navigation in the editor.** Go to definition, hover for types and docs,
  find all references, and parameter hints while typing a call now work in the
  editor for TypeScript, JavaScript, and Python. The language server for a file's
  language starts the first time you use it and unloads when idle, so it does not
  hold memory while you are not navigating.
- **Search your past conversations by content.** The search box in both Chat and
  Code mode now searches the full text of your previous messages, not just titles,
  and includes archived sessions. Results show a snippet with the match.
- **Verification badges in the editor gutter.** After the agent proposes changes,
  each changed file shows a pass, warning, or fail badge in the gutter from the
  quality checks, with per-line notes when code review is turned on. Badges appear
  on the proposed diff before you keep or discard it.
- **A live view of what is loaded on your GPU.** A status strip shows the resident
  model, free memory, and whether a second model would fit, so you can see the
  headroom before loading one.
- **Recent-projects welcome screen and per-project settings.** A richer project
  picker with recent projects, pinning, search, and the current branch inline.
  Each project can set its own default model. Opening a folder now asks whether to
  trust it before applying that folder's saved configuration, which closes a hole
  where a repository's committed config could apply automatically.
- **Per-project air-gap.** A project can be marked to stay offline regardless of
  the global setting. Its tool calls, shell, Git and GitHub automation, and cloud
  model access are held local. A project can tighten to offline but can never
  loosen a globally offline setting.
- **Side conversation while you code.** In Code mode you can open a side chat that
  reads your current session read-only and runs a model you choose, for a quick
  second thread without disturbing the main run. One result can be handed back
  into your main conversation.
- **Run a batch of tasks overnight.** Queue several tasks to run on your own GPU
  while you are away. Each runs verified in its own isolated copy of the project,
  and you get one scorecard summary in the morning with per-task apply and discard.
- **Recovery help when a run gets stuck.** When a run stalls, crashes, or hangs, a
  small local model can diagnose why and suggest how to unstick it, and it works
  even with no network. Off by default.
- **Bring agents that use the open Agent Skills format.** Import a skill package
  that uses the shared `SKILL.md` format. Imported skills are body-only by default,
  do not auto-run on their own, and any tool or command permissions they carry are
  shown for you to approve before they take effect.
- **Ask a local model for a second opinion mid-run** (`consult_mixture`) and
  **delegate a bounded subtask to a local sub-agent** (`delegate_subtask`). The
  sub-agent works in an isolated copy of the project and its changes are quality
  checked before they can touch your files. Both are off by default and can be
  turned on in settings.
- **Faster type-checking (experimental).** An optional setting swaps the
  TypeScript check used during verification for a faster native compiler. Off by
  default.
- **New models in the picker.** Kimi K3 (1M context), the GPT-5.6 family (Sol,
  Terra, and Luna), and Qwen3.7 Max are now available for the providers that
  serve them, including OpenRouter.
- **A dedicated QEL settings tab.** The quality-check controls moved out of
  Experimental into their own tab, with clear toggles for execution proofs,
  modification proofs, the semantic and answer-grounding judges, post-loop code
  review, contract-aware review, and the experimental type-check gate.

### Changed
- **Spend caps now cover every place a cloud call can happen.** Background runs,
  overnight batches, Git assist, and autocomplete all count toward your spend cap
  and appear in the cost view, closing gaps where some cloud usage was not being
  recorded.
- **More accurate GPU fit checks for parallel runs.** The limit on how many
  sessions can run at once now measures free memory rather than total, and
  understands that a shared local server uses one set of weights across slots, so
  it stops over-committing memory.
- **The side conversation now lives in Code mode**, where a second thread while
  you work is useful, and its button sits next to the Code panel toggle.

### Fixed
- **Local Qwen, DeepSeek-R1, and QwQ models no longer get stuck repeating
  themselves.** These model families document that low-temperature decoding
  causes repetition loops, but unlisted local tags were running at a low
  fallback temperature. Every affected family now uses its vendor's
  recommended sampling (Qwen3.x and Qwen2.5 at 0.7, DeepSeek-R1 and QwQ at
  0.6, Nemotron Nano at 0.6, Gemma and gpt-oss at 1.0).
- **Go to definition no longer opens a duplicate tab** when the target is a file
  you already have open (Windows).
- **Empty chat view fills the window** instead of leaving blank space below the
  composer.
- **Diagnostics settle before they are read**, so a language server that reports
  progressively no longer briefly shows a stale error after a clean edit.
- **The side conversation no longer crashes** when opened on a session that has no
  side conversations yet, and its buttons now match the rest of the app.
- **The API server meters and caps cloud spend on its streaming and agentic
  endpoints.** Previously a client using either of those could run cloud usage
  that was not counted toward your spend cap or shown in the cost view.
- **Web fetch blocks a wider set of private and loopback addresses**, closing gaps
  that could reach local-only services.
- **File edits requested through an alternate operation name are gated behind
  approval** like any other write.
- **A side conversation stays strictly read-only** and can no longer trigger
  project shell commands through verification.
- **Editor language-server messages with non-ASCII characters** (accents, emoji,
  CJK) no longer corrupt the language-server channel.
- **Applying a parallel run keeps uncommitted work** in its isolated copy and no
  longer sweeps in unrelated local changes.
- **Local model context sizing is safer when a model shares GPU slots**, avoiding
  an out-of-memory loop on load.
- **Side conversation replies no longer show leftover tool text.** A smaller
  model could leak a fragment of raw tool syntax at the end of a reply; it is
  now cleaned before display.
- **The side conversation now sees your open project.** It answered "no folder
  is attached" even with a project open because the open project's path was
  never handed to it. Its bubbles also now match Chat mode's styling.
- **Air-gap now also blocks embeddings to a non-local Ollama address.** If your
  Ollama URL points at another machine, air-gap mode refuses to send document
  text there (layer 17).
- **Cloud-routed inline fixes in the editor now count toward your spend cap**
  and appear in the cost view.
- **More local model families now use their vendor's recommended sampling.**
  Llama 3.1 through 4 (0.6), Devstral (0.15), and GLM-Z1 (0.6), each confirmed
  against the vendor's own published configuration.
- **A folder of smaller fixes from the hardening audit:** memory recovery no
  longer drops user scoping or embeddings, base URLs with embedded credentials
  are rejected on save, a far-future scheduled batch fires at the right time,
  a deferred batch item now actually waits out its backoff, mixture reference
  drafts cannot forge section delimiters, failed delegations no longer consume
  the retry budget, quality-report text is credential-scrubbed, and worktree
  cleanup finds projects whose names contain special characters.
- **Startup no longer misreports local model context limits.** Model context
  was computed before GPU memory detection finished, logging false warnings
  and briefly clamping context sizes on capable hardware.


---

## [1.0.0-beta.31.10.5] - 2026-07-10

### Fixed
- **Smaller models now apply code fixes instead of just describing them.** On a
  bug-fix or edit task, a smaller model would often write out a correct-looking
  explanation of the change but never actually edit the file, leaving you with
  nothing. Several causes are fixed together: the built-in Debug, Generate, Test,
  Refactor, and Perf skills were accidentally hiding the precise-edit tool from
  the model; edits are now matched more forgivingly so a minor whitespace
  difference in the target text no longer fails the edit outright; and when a run
  finishes with an explanation but no edit, the agent is prompted once to apply
  the change.
- **Agent runs on a cloud-hosted model are no longer cut short.** The five-minute
  safety limit was tuned for fast local models. A capable model served over a
  cloud provider is slower per step because of network round-trips, and long runs
  were being stopped mid-task. Cloud-served runs now get the longer budget, and
  the time the agent spends waiting out a provider's rate limit no longer counts
  against that budget.
- **When an edit can't be placed, the error now shows the nearest matching lines**
  in the file, so it's clear what to target instead of a generic "not found".
- **Run and cost summaries now name the model that actually served the turn.**
  A run on one model could be labeled with a different provider default, which
  also mis-priced the per-message cost estimate.
- **More accurate task understanding and context handling.** Task requirements are
  now read from the project's own files (language and framework) instead of being
  guessed from the request wording, which fixes occasional misclassification.
  Several context-size calculations were corrected so long runs are measured and
  trimmed against the real window, and a reported-token quirk that made long runs
  look like they had overflowed (when they had not) is resolved.

### Changed
- **Local models start leaner.** A local model now starts with a smaller core set
  of tools and loads the rest on demand, rather than carrying every tool's full
  description from the first message. This frees up context on small models
  without changing what the model can ultimately do.

---

## [1.0.0-beta.31.10.4] - 2026-07-09

### Fixed
- **Terminal windows no longer pop up when an MCP server fails to start.** On
  Windows, a configured MCP server that couldn't start (for example a server
  whose runtime isn't installed) caused a visible console window to flash open
  every few seconds, forever. Two fixes: MCP server processes now start hidden,
  and a server that keeps failing now backs off and stops after five attempts,
  showing a clear "failed" status in Settings with the last error and a Connect
  button to retry manually.
- **Clicking a settings control no longer shoves the panel out of view.**
  Clicking an option in sections like FIM or Codebase Embeddings could shift the
  whole settings panel upward and leave a permanent block of empty space at the
  bottom, in both Chat and Code mode. The browser was scrolling a container that
  users can't scroll back; that container type can no longer be scrolled at all.
  Dropdown pickers in Settings were also moved out of the scrolling area so an
  open picker can't distort the panel's height.
- The diagnostics bundle now labels MCP servers by what they actually are
  (stdio, with or without network access) instead of a misleading "network"
  tag, and shows a clear "failed (gave up)" status.

### Added
- **Experimental: deferred tool loading for local models.** A new setting under
  Agent (`agent.deferred_tools`, off by default) sends the model a smaller core
  set of tools up front and lets it pull in the rest on demand, which trims the
  tokens spent on tool definitions every turn. It's off by default while we
  measure it, applies only to local models when set to "local only", and never
  changes which tools are allowed or how approvals work. If you don't turn it on,
  nothing changes.

### Internal
- Added a test harness for the agent loop: a record and replay rig, a cache-prefix
  stability gate, and a set of golden tasks seeded from real bugs, so a change
  that would break the agent's behavior or its prompt caching now fails in tests
  instead of shipping. No user-facing change.

---

## [1.0.0-beta.31.10.3] - 2026-07-09

### Added
- **The model catalog now updates on its own, no app update needed to get new
  models.** Bodega can pull a refreshed catalog (model list, hardware-fit info,
  and downloadable GGUF entries) from a single Bodega-hosted file. This is on by
  default and is a plain model list only: no telemetry, one fixed URL, and it
  never runs in air-gap mode. There's a clear toggle in Settings under Models to
  turn it off. When it's off, the Discover screens show a small note that new
  models won't appear until you turn it back on or install the next app update.
  You can still refresh manually with the button in Models.
- **July 2026 model refresh.** Added Grok 4.5 and GLM-5.2 to the cloud model
  list (with pricing), corrected the Kimi context length, and added two new
  local vision models, Qwen3-VL 32B and MiniCPM-V 4.5, with their image
  projector files so they download and run out of the box.
- **A cleaner way to browse models.** The Discover screens now group models by
  family (Qwen, Llama, Gemma, Mistral, DeepSeek, and so on) in collapsible
  sections, and each model card shows capability tags (tools, vision, thinking,
  FIM, draft, MoE) so you can see what a model does at a glance. Cards now also
  tell you whether a model fits your GPU, and which quantization fits best,
  before you start the download rather than after.
- **Vision companion picker.** You can now pick which vision model pairs with
  your text model for image questions. The picker lists your installed
  vision-capable models with their engine, VRAM cost, and how the swap works
  (Ollama swaps internally, llama.cpp hot-swaps the server process), and
  remembers your choice as the default pairing.
- **Manage your background worktrees.** The Fleet panel now shows worktrees
  queued for automatic cleanup, lets you confirm or cancel each one, and warns
  you when they start taking up significant disk space. A new "Manage
  worktrees" view lists per-session storage and lets you reclaim disk on demand.
- **Answers to questions about your code now get a groundedness check.** When
  Bodega answers a question after reading your files, it quietly checks that the
  answer is actually backed by what it read, flagging a reply that cites a file
  it never opened, or that's too thin to trust. It's advisory only: you get a
  note, never a blocked answer.
- **Modification tasks can use your project's own lint and tests as proof.** When
  Bodega changes or fixes existing code, it can run your project's own lint and
  test scripts once at the end as evidence the change holds up. A clean pass
  strengthens the result, a real failure is surfaced as an advisory note rather
  than silently ignored. On by default, and does nothing when there's no lint or
  test script to run.
- **Optional second-opinion review of changes.** You can turn on a review that
  asks a second model to check a change against what the task actually asked for,
  as an extra set of eyes on the diff. It's advisory only and off by default.
- **Agent-driven automation runs now show up in run history** alongside scheduled
  loop runs, so a run started by an agent is no longer invisible after it finishes.

### Changed
- **Reasoning effort controls now match what each model actually supports.** The
  effort control (off through max) now appears for exactly the models that honor
  it, with the right levels per model, whether that model is your primary or one
  you picked for a single panel. DeepSeek now maps to its real off/high/max
  tiers, Gemini 3.x uses its new thinking levels instead of the retired numeric
  budget, and turning reasoning off on an Ollama model now actually turns it off.
  Models that don't support graded effort show a simple thinking on/off toggle
  instead of levels that did nothing.
- **The Model Roles settings were rebuilt.** Each role is now a proper model
  picker card that shows the provider, what an empty role falls back to, and a
  one-click reset, with a real save button that knows when you have unsaved
  changes.
- **Status colors are readable in light themes.** Success, warning, error, and
  info text and banners were re-tuned so they meet contrast guidelines across all
  four themes, fixing the green-on-green and similar washed-out cases in the light
  themes. A test now guards these color pairs so they can't regress.
- **Onboarding was refreshed to match the current look.** The first-run flow now
  uses the current color tokens throughout, so the light themes render correctly,
  and some dead onboarding code was removed.
- **Verification messages now tell you what's blocking and what's just advice.**
  Verification results clearly separate blocking issues from advisory notes, and
  long diagnostic output is capped so a report stays readable instead of flooding
  the panel.
- **Drift Radar is now a single surface.** The separate Drift panel was folded
  into the Debug panel's Drift tab, and when an oracle has legitimately changed
  you now get a one-click Retire action to clear the noise.
- **The diagnostics bundle now captures more.** Export Diagnostics now includes
  model catalog state, the vision pairing, worktree cleanup and disk state, and a
  summary of recent verification results, and it now catches uncaught errors and
  promise rejections that previously slipped past the report.

### Fixed
- **Local models on llama.cpp could fail to respond.** Some models (Qwen3.5 among
  them) use a strict chat template that rejected how Bodega packaged the prompt,
  so every message came back as a failure. The prompt is now packaged in a form
  those templates accept, without changing the caching behavior that keeps local
  models fast.
- **Switching your primary provider could leave the app in a split state.** After
  changing your primary model provider, a chat could still be routed to the old
  provider (for example a local GGUF sent to Ollama, which then reported the model
  as missing), a provider card could show "offline" while its models were clearly
  connected, and the agent panel header could keep showing the previous model
  after a swap. Provider state is now reconciled in one place so these stay in
  sync, and a downloaded GGUF is always recognized as a local model.
- **Chat sends are more reliable.** A message retried after a dropped connection
  no longer creates a duplicate, and a queued follow-up message can no longer be
  silently dropped when you cancel or when the app restarts mid-run.
- **The model picker no longer lists other providers' models.** A single-vendor
  cloud provider (for example Qwen) could surface unrelated models returned by the
  provider's API. Those are now filtered out.
- **Local model context sizing is now regression-proofed.** The context size a
  model card shows now matches what the inspector shows, and the default context
  window is derived from your card's total VRAM rather than whatever happened to
  be free at the moment, with tests so it can't quietly break again.

---

## [1.0.0-beta.31.10.2] - 2026-07-08

### Added
- **Qwen3-Coder-Next 80B is now downloadable.** Qwen's flagship open code model
  ships as a multi-file (sharded) GGUF, which the downloader couldn't handle
  before. Bodega now fetches every shard, resumes each one independently if the
  download drops, and loads them as a single model — and removing the model
  deletes all of its shards, not just the first. Q4_K_M is about 45 GB across
  four files (needs a 48GB+ VRAM card).
- **Two new models in the catalog:** SmolLM3-3B, a compact reasoning + tool
  model that runs on low-VRAM machines, and Mistral Small 3.2 24B, an updated
  build with better instruction-following than 3.1. Both were verified against
  HuggingFace so their downloads resolve.

### Fixed
- **GGUF downloads no longer fail with a false "size mismatch."** A complete,
  valid download could be rejected because it was checked against a size
  estimated from the catalog's rounded GB figure rather than the real file.
  Downloads are now validated against the size the server actually reports, so
  anything fully received finalizes. This is what was blocking the Qwen3.6 27B
  download.
- **Cloud providers you set up now show up in the model picker.** Pasting an API
  key is meant to switch its provider on, but when the provider's entry didn't
  exist yet — common right after a settings reset — the key saved without
  enabling it, so providers like Anthropic, OpenRouter, DeepSeek, and Qwen
  stayed hidden even though their keys were stored. A saved key now reliably
  enables its provider.
- **Local model context is no longer capped far too low.** A model's default
  context window was sized against however much VRAM happened to be free at the
  moment it loaded. With a browser or other apps open, that could shrink a 27B
  model on a 32 GB card to roughly 8K tokens and leave it stuck there for the
  session. Context is now sized against the card's total VRAM, and the value on
  the model card matches what the running model actually uses.
- **The moondream2 vision model downloads again.** Its GGUF files moved to a new
  HuggingFace repository; the catalog now points at the right one.

## [1.0.0-beta.31.10.1] - 2026-07-07

### Fixed
- **The Linux app now runs on mainstream distributions.** The bundled SQLite
  native module required GLIBC 2.38, so the Linux AppImage crashed on startup
  on every Linux older than Ubuntu 24.04 — Ubuntu 22.04 LTS, Debian 11/12,
  RHEL 8/9, Amazon Linux. The whole app was affected (every launch loads the
  database first). The Linux build now runs on an older-GLIBC runner (2.35)
  and compiles the backend SQLite from source, with a build-time guard that
  fails the release if it still needs a too-new GLIBC. Windows and macOS were
  never affected.
- **Questions can no longer end as a tool work-log.** Asking an orientation
  question ("what does this project do?") could end with the model going
  silent after its tool calls — and the reply became "Done! Here's what I
  did:" plus a list of tool names, answering nothing. Every retry nudge was
  gated to command-style tasks, so question turns had no rescue. Now any turn
  that ends with tool activity but no text gets one final tools-off model
  call to answer from what was gathered (the same rescue the iteration-cap
  exit already had), and the last-resort static summary now leads with actual
  findings — the codebase-map answer rides along, file reads collapse into
  one named list, and repeated tool names dedupe.

## [1.0.0-beta.31.10] - 2026-07-06

### Added
- **`bodega://models/huggingface/{owner}/{repo}` deep links.** The `bodega://`
  protocol handler routes its first real destination: a HuggingFace model link
  opens Settings → Models → Discover with the repo ready to download —
  llama.cpp primary auto-searches the GGUF browser for it (the disclosure pops
  open); Ollama primary prefills the custom pull input with the `hf.co/` form;
  air-gapped machines get a clear toast instead of a silent no-op. This is the
  "Use this model" flow for the HuggingFace Local Apps listing. Deep-link
  payloads are strictly validated in the main process before anything reaches
  the renderer (raw-pathname matching, bounded length, charset-locked), and
  unknown query params are tolerated so future hints (e.g. `?file=`) won't
  break older builds. The parser is unit-tested against injection shapes.

### Fixed
- **Cold-start deep links no longer vanish on Windows/Linux.** A `bodega://`
  click with the app closed launched it to nothing: the initial-launch argv
  scan only extracted file paths, so protocol URLs were handled only when a
  copy was already running (`second-instance`). The launch argv is now scanned
  for protocol URLs too, and the routed action queues until the renderer has
  finished loading — same pattern the file-association path already used.
- **Packaged macOS/Linux builds never registered the `bodega://` protocol.**
  The electron-builder config lacked the `protocols` declaration, so packaged
  macOS builds had no `CFBundleURLTypes` (runtime registration only covers
  dev-mode) and Linux desktop entries had no `x-scheme-handler` MimeType —
  deep links errored at the OS level before the app ever saw them. Declared;
  Windows was already covered by runtime registration.

---

## [1.0.0-beta.31.9.1] - 2026-07-06

### Added
- **ACP `initialize` advertises `authMethods`.** The agentic-coding-protocol
  handshake omitted the field entirely; Bodega is local-first (no account, no
  sign-in), so it now emits an explicit empty `authMethods: []` — a complete,
  spec-correct response for a strict ACP client or the agent registry validator,
  which makes the terminal surface registry-ready.

### Fixed
- **The `gpt-oss` phantom model is gone.** `OllamaProvider` booted with a
  hardcoded `gpt-oss` placeholder default; any code path that read the default
  before a healer replaced it reported — or requested — a model the user never
  installed (agentic run summaries mislabeled `gpt-oss` while another model
  actually served the tokens). The placeholder is removed: the default is empty
  until lazily resolved from the *installed* model list via the embedder-aware
  picker (never an embedding model, never a name that isn't present), and the
  "no model selected" guidance stays for a machine with nothing installed.
- **Local model auto-selection never lands on an embedder.** The reconfigure
  healer took the alphabetically-first installed model, so a machine whose first
  model was an embedder (e.g. `qwen3-embedding:4b`) got an un-chattable default
  and every request failed "does not support chat." It now uses the same
  embedder-aware curated picker.
- **File edits survive CRLF checkouts.** `str_replace` matched the old string
  byte-for-byte, but a model cannot see carriage returns in the content it
  reads, so on a CRLF working tree (Windows / git autocrlf) every edit failed
  "old_string not found" until the run gave up. Matching is now line-ending
  tolerant and preserves the file's own EOL style in the replacement.
- **QEL no longer fails correct removals.** A "remove X" / "delete X" task
  extracted the named construct and then verified it was still *present* —
  demanding the continued existence of the thing the task asked to eliminate,
  an unsatisfiable check that thrashed the repair loop and mislabeled correct
  work as failed. Verification now derives the expectation from the verb
  (add/change → present, remove/delete → absent), and `rename X to Y` verifies
  the new name instead of the old one.
- **Big local models get their real context window.** The VRAM-safe window
  estimator sized models against an *instantaneous free-VRAM snapshot* — so a
  27B sized while an 8B was still resident got an 8,356-token window on a
  32GB card (the correct post-eviction answer is ~19K), the bad-moment probe
  was memoized for the whole session, and a model sized while itself resident
  had its weights subtracted twice. The estimator now sizes for the post-load
  world (total VRAM minus a fixed reserve — Ollama evicts the resident model
  on load); instantaneous free VRAM still drives the can-these-coexist check,
  where it belongs.
- **A degraded cloud model list can no longer override your model choice.**
  When a rate-limited provider returned a partial `/models` response, the
  auto-default healer treated the user's explicitly-chosen model as "not
  available" and silently replaced it with a curated pick (a rate-limited Kimi
  session was flipped to gpt-4o). Not-in-list healing now applies only to
  local providers, whose lists are complete enumerations; on cloud presets a
  user-set model always wins. Empty-response summaries also now carry real
  results for diagnostics and test runs instead of "Used get_diagnostics".

## [1.0.0-beta.31.9] - 2026-07-06

### Added
- **The docs hub covers the CLI.** A new "Bodega One Code (CLI)" section in the
  in-app docs: what the terminal surface is, install one-liners for all three
  platforms, first-run setup, the command table, headless/CI behavior, and how
  the CLI shares this app's providers, settings, and sessions through the
  family data dir.

### Fixed
- **Voice transcription keeps the uploaded audio's real format.** The
  speech-to-text route ignored the upload's declared Content-Type and stamped
  every temp file `.webm`, so an OGG voice note (how Telegram and Discord voice
  messages arrive via the bodega-agent, gap A3b) reached extension-sniffing STT
  engines mislabeled. The route now reads the part's Content-Type, writes a
  matching temp extension, and passes the MIME through to the STT service. The
  desktop app's own recorder (webm, no declared part type) behaves exactly as
  before.
- **The shared database is now safe when two Bodega apps run at once** (S1
  follow-through — four concurrency fixes). The family-shared data dir means the
  IDE backend and the Telegram-agent backend can open the *same* `bodega.db` as
  separate OS processes, which surfaced four check-then-act races (all found by
  battle-testing, each proven with a true two-process stress test and pinned by
  regression tests):
  1. *Lost writes* — no `busy_timeout` was set, so a second concurrent writer's
     write was silently dropped (~0.5% under contention). Now `busy_timeout=5000`,
     set as the FIRST pragma: 0 losses in 1800 contended writes.
  2. *Stranded first-session data* — the migration race-loser did a single
     done-marker check and fell back to the legacy dir, stranding its writes
     there. It now polls (bounded) for the winner, matching the Go CLI.
  3. *Clobbered encryption key* — two fresh backends could both generate
     `.bodega-cipher-key`, the loser overwriting the winner's key and making its
     already-encrypted secrets permanently undecryptable (8/8 races corrupted).
     Key creation is now atomic (exclusive-create; a race loser adopts the
     winner's key): 0/8.
  4. *Boot crash on concurrent schema init* — two backends booting together both
     ran the check-then-ALTER migrations; the second crashed with "duplicate
     column name" (6/6), and the fresh-DB WAL conversion could also fail with an
     immediate SQLITE_BUSY (~1 in 6). The entire open + schema sequence is now
     serialised across processes by a lockfile (stale-TTL reclaimed, bounded
     wait, fail-open so a lock problem can never block boot): 8/8 clean
     two-process first boots.
- **Family-shared canonical data dir** (gap S1, desktop half). The desktop IDE,
  CLI, and agent historically resolved *three different* data dirs, so a provider
  key / MCP server / session set in one was invisible to the others. The IDE now
  resolves the neutral canonical location shared with the CLI + agent
  (`%LOCALAPPDATA%\BodegaOne` on Windows, `~/Library/Application Support/BodegaOne`
  on macOS, `${XDG_DATA_HOME:-~/.local/share}/bodegaone` on Linux), wired through
  every main-process anchor (backend spawn env, diagnostics resolver, worktree
  base, air-gap.lock). Migration is strictly safe — **copy-not-move** (a legacy
  dir is never touched), auto-migrating only the unambiguous single-legacy-source
  case; when multiple populated legacy dirs exist it keeps using the IDE's own
  `userData` dir (zero regression, the 48MB store stays put) and drops a
  `.consolidation-pending.json` marker for a deliberate consolidation step. A hot
  source (an app actively writing its DB) defers migration. `BODEGA_USER_DATA_DIR`
  still overrides everything (QA scripts / demo-fresh-install). Behaviourally
  identical to the CLI's Go implementation; 6 unit tests mirror the CLI's safety
  suite. NOTE: needs an Electron live-smoke before a main release — the migration
  logic is unit-verified but the full app-boot integration is not yet live-tested.
- **Background commands can no longer leave orphaned processes behind.** A dev
  server started in the background could survive closing the app (its helper
  process escaped the shutdown cleanup) and keep a port occupied — which could
  then block the local model server from starting. Shutdown now verifies the
  whole process tree is gone.
- **Background commands can't take over Bodega's own ports.** A backgrounded
  command that tries to serve on a port Bodega itself uses (backend, local model
  server, embeddings) is refused up front with a suggestion to pick another port.
- **A port conflict is reported as a port conflict.** When the local model server
  can't start because its port is taken, the error now says so — instead of the
  misleading "context window too high" message.

### Security
- **`--deny-tools` can no longer be evaded by tool-name aliases.** The per-request
  tool deny list matched the exact name the model emitted, but the executor
  resolves aliases before running a tool (`bash`/`run`/`git`/`terminal` → shell,
  `write_file`/`create_file` → file_system, `edit_file` → str_replace). So a denied
  `shell` was bypassed by emitting `bash`, and a denied `file_system` by emitting
  `write_file`. The deny check now canonicalizes both the incoming name and each
  deny-list entry. This is a shared-backend fix: it protects every surface that
  runs the agentic loop — the desktop app, the `bodega` CLI (`--deny-tools`), and
  the headless agent. Found via live battle-testing.

## [1.0.0-beta.31.8.1] - 2026-07-02

### Fixed
- **"Open the preview" no longer opens Bodega's own model server.** Asking the
  assistant to open the preview could land on the local model server's built-in
  chat page (a "Hello there" screen on localhost:8080) instead of the project,
  because the dev-server port scan counted Bodega's own servers as dev servers.
- **The dev-server scan skips Bodega's own ports.** Auto-detection no longer
  considers the ports Bodega itself listens on (the backend, the local model
  server, and the embeddings server), so it can only ever find a real dev server
  or fall back to serving the project folder.
- **Asking to preview one of Bodega's own addresses is corrected, not obeyed.**
  If the assistant is told (or guesses) an address that belongs to Bodega's own
  servers, the preview opens the actual project instead and says why.
- **Preview navigation to Bodega's own servers is blocked with guidance.**
  Direct navigation to one of those addresses now returns a clear pointer to the
  correct way to open the project preview instead of showing the wrong page.

## [1.0.0-beta.31.8] - 2026-07-02

### Added
- **Optional quality gate for Mixture reference drafts.** With Bodega Mixture
  on, a weak or off-topic reference draft could still be handed to the
  aggregator and drag down the final answer. A new opt-in setting
  (`mixture.qel_gate`) screens each reference draft before synthesis and drops
  the ones that fail, so the aggregator only works from drafts worth keeping.
  Off by default.

### Changed
- **Faster startup and snappier sessions.** Project file listings are now
  cached instead of re-scanned on every request, session messages load through
  a new database index, non-essential startup checks are deferred until after
  the app is up, and the agentic loop does less redundant work per step.
- **External clients can now wait for the full toolset.** The backend used for
  editor integrations signals a second "ready" once MCP servers and skills have
  finished loading, so an external client can tell when the complete toolset is
  available instead of connecting into a partial one.

### Fixed
- **Cloud Boost reasoning settings no longer depend on the local base model.**
  The remaining places where a boosted cloud run could inherit settings from
  the local base model (such as reasoning behavior) now consistently use the
  cloud model that is actually running.
- **Building a project from scratch no longer triggers helper skills meant for
  existing code.** A "build X" request in an empty folder could auto-activate a
  skill written for modifying an existing codebase, sending the model down the
  wrong path. Skills now check the shape of the task before activating.
- **Verification reads routes and function names more accurately.** The
  contract extractor that checks a finished task against your request no longer
  mistakes HTTP verbs for function names or trips on trailing punctuation in
  route paths, so verification scores fewer false misses.

### Security
- **Air-gap mode blocks more ways to reach the network from the shell.**
  Additional download and scripting tools that can fetch remote content
  (including certutil, bitsadmin, perl, ruby, and php) are now blocked while
  air-gapped, and the check is resistant to quoting tricks that previously
  slipped past it.
- **Hooks that rewrite a command can no longer bypass approval.** If a
  configured hook rewrites a tool call into something dangerous, the rewritten
  command is re-checked with a fresh security context and always requires your
  approval; web fetches and searches also always prompt in Ask mode.
- **Concurrent streams on one session are properly serialized.** Two responses
  can no longer run against the same session at once, and a session that is
  actively streaming can no longer be deleted out from under itself.

---

## [1.0.0-beta.31.7] - 2026-07-01

### Added
- **Claude Sonnet 5 and Claude Fable 5.** Sonnet 5 is a new Anthropic flagship
  with a 1M-token context window. Fable 5 is back after being temporarily
  withdrawn from general availability; both are available for Anthropic,
  OpenRouter, and Cloud Boost.
- **Faster repeated turns on local models.** llama.cpp and Ollama now reuse the
  prompt cache and keep the model warm between turns, so follow-up messages in a
  conversation start responding sooner.

### Fixed
- **Cloud Boost no longer stalls on long tasks with Claude models.** Running a
  Claude model (Sonnet 5, Fable 5, or Opus) through Cloud Boost while a local
  model was the base could make it stall and return nothing on longer requests,
  because the boosted request was mis-handled as if it were a local model. Cloud
  Boost now treats these as cloud models, so they use native thinking and respond
  normally.
- **Capable models keep their full step budget on long tasks.** A few recoverable
  tool errors part-way through a long task could downgrade a strong model to a
  short step limit for the rest of the session; strong models now keep their full
  budget (the safety limit still applies to smaller local models).
- **A from-scratch build no longer runs to the step limit after it is done.** On
  a "build X" request, a code-refactoring helper could switch on by mistake and
  push the model to keep writing and running extra verification files it had
  nowhere valid to save, using up the remaining steps after the work was already
  finished. Build requests no longer trigger that helper.
- **"Open the preview" no longer errors with a text-only local model.** Asking a
  local model without vision support to screenshot the preview could fail with a
  server error, and the preview could open the wrong page. Screenshots now fall
  back gracefully for text-only models (the model is pointed to the captured image
  instead), and the assistant is steered to open the correct preview URL.
- **Switching local models is more reliable.** Starting, stopping, swapping, and
  recovering the local model server are now serialized, so rapid model switches
  (including vision hand-offs) no longer race or crash.
- **The app still starts if its license file can't be read.** An unreadable or
  undecryptable license now falls back to unlicensed instead of stopping the app
  from booting.
- **The Mixture progress card always finishes.** After a Mixture turn ended, the
  progress card could keep spinning; it now resolves to a final state.

---

## [1.0.0-beta.31.6] - 2026-06-29

### Added
- **Bodega Mixture (Mixture-of-Agents).** A new optional engine that runs several
  "reference" models in parallel on your turn, then has one "aggregator" model
  synthesize the single response you see. Enable it in Settings under Models, then
  pick "Mixture" from the model picker. Reference models run with no tools and the
  conversation text only; the aggregator owns all tools and writes the reply. Local
  reference models are near-free, so an all-local mixture costs about one cloud
  call, and an optional QEL quality gate can verify the synthesized output. Off by
  default; all `mixture.*` settings are global-only. Honors air-gap (cloud
  references are dropped when offline) and falls back to a normal single-model turn
  when fewer than two references are available.
- **Preview static sites and games without a dev server.** The in-app Preview
  could only attach to a running dev server like Vite or Next, so a plain
  project (a bare `index.html` with script files and no build step, like a
  generated HTML5 game) had no way to preview and just told you to start a dev
  server. Now, when no dev server is running and the project has an `index.html`
  at its root, Bodega serves the folder locally and opens it in the Preview
  panel. Ask the agent to "open the preview", or use the "Preview this folder"
  button in the Preview panel's empty state. Served on loopback only (air-gap
  safe), with dotfiles withheld.

### Fixed
- **Long Cloud Boost builds no longer stop early at a five-minute mark.** A
  fixed five-minute time limit on the agentic loop could cut off a capable cloud
  model (such as Opus via Cloud Boost) partway through a long build, after which
  a "continue" was mis-read as a fresh, simple task and capped at twelve steps.
  The loop's time budget now scales with the model's capability tier (longer for
  large and boosted or cloud models), and a "continue" on an in-progress task
  stays in the full agentic lane with the full step budget.
- **Local models no longer get stuck at a 4,096-token window.** A local GGUF with
  no embedded context-length metadata had its window silently clamped to 4,096
  even on a high-VRAM card, and the output-token reservation (sized from a
  separate, larger estimate) could then exceed that window, so every message,
  even a one-word "yo", was rejected as "too long". Bodega now sizes an
  unknown-metadata model's window from available VRAM instead of collapsing to
  4,096, keeps a custom `--ctx-size` in sync between the server and the token
  budget, and never lets the output reservation exceed the window. Restart the
  app to pick up the corrected window.
- **Mixture reference models can now be added from the Settings UI.** The Mixture
  editor saved the reference-model list in a format the backend rejected, so
  clicking "Add" in Settings silently did nothing and an already-configured list
  never rendered its rows. Adding, removing, and reading reference models now
  works (the picker dropdown also groups by provider, shows your local Ollama and
  llama.cpp models, and uses clean model names).

---

## [1.0.0-beta.31.5] - 2026-06-26

Security and reliability rollup from two app-wide bug-hunt sweeps: closes a set
of approval-gate, air-gap, and DNS-rebinding gaps, prevents an index-wipe and a
few silent data-loss paths, and fixes a batch of provider, streaming, and
concurrency bugs found across the app, plus a set of Cloud Boost fixes. No
feature changes.

### Cloud Boost & context
- **Cloud Boost no longer crashes partway through a task.** When Boost (a cloud
  model like Opus) drove a long code build, the conversation could be trimmed in
  a way that left a tool result without its matching tool call, which the cloud
  API rejected and stopped the run. The history sent to the cloud model is now
  always structurally valid, and trimming keeps tool call/result pairs together.
- **The context meter reflects the model that's actually running.** With Cloud
  Boost on, the meter stayed on your local model and its (small) window, so a
  short conversation could read as over 100% full and force needless
  compaction. It now shows the active boost model's real window, both while the
  run is in progress and in the inspector, even when a local llama.cpp model is
  set as the primary.
- **A run that stops because your provider is out of credit now tells you why.**
  When a cloud run ran out of API credit (or the key was invalid or revoked, or
  the provider was disabled) partway through a task, it showed a generic "reached
  the iteration limit" message with no error, so it looked like the agent gave up
  on its own. It now surfaces the actual provider error, and the "iteration
  limit" message only appears when the run genuinely used all of its steps.
- **The max-iterations setting is honored on capable models.** A stale internal
  cap limited the Agent panel to 25 steps regardless of your max-iterations
  setting, which only bit higher-capability cloud/boost models (local models
  finished under the cap, so it was invisible there). Capable runs now use your
  configured budget (default 50) instead of stopping early at 25.
- **The local model's context size is read correctly.** A custom `--ctx-size` in
  a model's llama.cpp arguments is now used as the window the app budgets
  against, so the meter matches what the server actually runs instead of falling
  back to a small default.
- **Clean model names in the Cloud Boost picker and the Spending dashboard.**
  They now use the same readable names as every other picker (for example
  "Claude Opus 4.8 (1M context)" instead of the raw id).
- **The Spending chart labels today on its axis and no longer clips a tall day.**
  The daily-spend chart drew the most recent day's bar but only labeled every
  fifth day, so the axis looked like it ended a few days early; the latest day is
  now always labeled. A day with a large total also drew a bar that ran off the
  top edge and overlapped the legend, so the chart now reserves room above the
  bars for the top axis label and the legend.

### Security
- **MCP tools now go through the approval prompt in Ask and Plan modes.** A tool
  from a connected MCP server could previously run without a per-tool prompt,
  even one that changes external state (opening a pull request, sending a
  message). They now ask first, the same as the built-in write tools.
- **Air-gap mode now also covers voice transcription, inline code completion, and
  the goal reviewer.** Each of these could reach a remote endpoint when pointed
  at a non-local address; with air-gap on they now refuse anything that isn't
  local, like the chat and embedding paths already did.
- **Web fetch is hardened against DNS rebinding.** It now validates every address
  a hostname resolves to and refuses when it can't confirm the target is public,
  instead of trusting a single lookup.
- **The local API and WebSocket reject requests whose Host header isn't the
  loopback address**, closing a DNS-rebinding path that could let a web page in
  your browser reach the local server.
- **Prompts supplied by a shared project config are sanitized before use.** A
  committed project config can set panel prompts; those are now stripped of
  markup that could forge instructions in the system prompt.
- **Output credential scanning runs before long output is truncated**, so a
  secret split across the truncation boundary can no longer slip through.
- **Hunk staging (git apply) is confined to your allowed project folders**, the
  same sandbox every other git operation already used.

### Fixed

#### Code mode
- **Undo now removes files (and folders) the agent created, instead of emptying them.**
  Reverting an agent-created file in the Changes panel or the task-complete card
  used to write the file's "before" content back, but for a brand-new file that
  content is empty, so undo left a 0-byte file and the new folders behind. Undo
  now deletes a created file and prunes the now-empty folders it created (a
  modified file is still restored to its original content). Folder pruning only
  ever removes truly-empty directories and never touches anything outside your
  project, so no other files can be affected.

#### Models & providers
- **GGUF files dropped into the models folder now show up in the app.** Bodega's
  model registry only tracked GGUFs added through a Model Hub download or an
  explicit sideload, so a `.gguf` copied straight into the llama.cpp models folder
  never appeared in Settings → llama.cpp models and couldn't be loaded by name.
  Bodega now scans the models folder on startup and whenever you open the model
  list, auto-registering any new GGUFs it finds (embedding and vision-projector
  files are skipped). A manual rescan is also available so a just-copied file
  shows up without restarting.
- **Truncated model downloads are now caught instead of silently failing later.**
  A GGUF download that ended short of its expected size could be marked finished,
  then fail to load with a confusing error. Downloads now verify the final size,
  clean up the partial file on failure, and resume from where they stopped when
  you retry.
- **Switching models mid-conversation now actually loads the new model.** A model
  swap requested while a response was streaming could leave the old model in
  place. The swap now applies reliably.
- **Sideloaded GGUFs survive a provider switch.** A model you added by hand could
  drop out of the list after switching providers and back. It now persists.
- **A model deleted from disk is flagged instead of failing on load.** If the
  file behind a registered model is gone, the list now marks it as missing rather
  than letting you pick it and hit an opaque load error.
- **A model that needs a vision projector but is missing one degrades gracefully.**
  Instead of a hard failure, you get a clear message that the projector file is
  required.
- **Failed model pulls no longer report success.** A download that errored or was
  cancelled could still surface as "ready." The real terminal status is now
  reported.
- **The embedding model is no longer picked as your chat model.** Embedding-only
  models (for example `text-embedding-*` and `Qwen3-Embedding-*`) are excluded
  from automatic chat-model selection.
- **Cloud providers that hit a network error stay visible with an error.** A
  transient connection failure could make a configured cloud provider vanish from
  the picker. It now stays listed and shows the error.
- **Testing a cloud API key tells transient failures apart from bad keys.** A
  "couldn't reach the provider" result now shows amber ("try again") instead of
  red "invalid key," so a network hiccup is not mistaken for a wrong key.

#### Sessions & background runs
- **Finished background and parallel runs no longer count against your session
  limit.** Completed runs were still being counted, which could lock you out of
  starting new ones until you restarted. They are now released when they finish.
- **A background run that errors surfaces the error.** A failed background or
  detached run could look stuck instead of reporting what went wrong. The error
  now reaches you.
- **Orphaned llama-server processes are cleaned up.** A llama-server left behind
  by a crashed or cancelled run is now terminated rather than holding the port.

#### Onboarding
- **First-run setup no longer finishes silently when the model never connects.**
  If the local model fails to come up during setup, you now get a warning instead
  of setup completing as if everything were ready.
- **A failed settings save during first-run is reported.** If saving your setup
  choices fails, you are told, rather than the flow completing with stale config.

#### Privacy
- **Air-gap mode blocks the embedding model from reaching a remote host at
  startup.** Boot no longer pre-warms a cloud embedding endpoint when air-gap is
  on.

#### Reliability
- **Large local models get more time to load before timing out.** Big GGUFs that
  legitimately take a while to load are no longer cut off early.
- **The llama.cpp models list shows an error with Retry when the backend is
  unreachable.** A failed fetch previously looked like a clean "no models
  installed" empty state.

#### Data safety
- **A transient failure to read your project folder no longer wipes the code
  index.** An empty scan caused by a locked or briefly-unavailable folder is
  treated as a probable read failure, not a signal to delete the index.
- **Deleting a session now stops its background run and removes its worktree**
  instead of leaving the run going and the folder orphaned.
- **A schema upgrade that rebuilds a table is now crash-safe**, so an interrupted
  upgrade can't lose run history.
- **Deleting a file outside the app no longer silently discards unsaved edits**
  in its open tab.
- **The code-search index skips a corrupt row instead of failing the whole
  search.**

#### Reliability & correctness
- **A background model load no longer overrides a provider you switched to while
  it was still loading.**
- **Empty (204) responses no longer surface as save failures.**
- **Progress streams (installs, downloads, model swaps) show an error instead of
  spinning forever when the request fails.**
- **The memory auto-extract off-switch now stops every auto-extraction path**, not
  just one of them.
- **A failed test run no longer reports as a clean pass** when the runner crashes
  without a results summary.
- **Cloud setup tells you when saving the API key failed** instead of showing a
  successful connection.
- **Per-provider model actions are correct:** the delete button only shows for
  providers that support local deletion, the code-completion model picker clears a
  stale model when you change provider, and session export uses the app's real
  backend port.
- **Smaller streaming and editor fixes:** duplicated text after a stream retry,
  the editor theme and language switches not taking effect, and the previous
  session's terminal output lingering after a switch.

#### Concurrency
- **Background runs can't start twice from a double-click or quick retry.**
- **Applying a parallel-run winner stops the other runs before cleaning up their
  folders**, avoiding corruption from a half-finished write.
- **Chat stream locks are ownership-checked** so a finishing stream can't release
  a newer one.
- **The follow-up message queue claims each message atomically**, so a queued
  message can't be processed twice.

## [1.0.0-beta.31.4] - 2026-06-25

Local-model and provider robustness — fixes for running custom OpenAI-compatible
servers (vLLM, llama-swap) and for how the agent recovers from tool slips on
smaller local models.

### Fixed

#### Models & providers
- **"Auto" could pick an embeddings model as the chat model.** The filter that
  keeps embedding-only models out of chat auto-selection missed size-suffixed
  names (`Qwen3-Embedding-0.6B`/`-4B`/`-8B`, `text-embedding-3-*`), so on a custom
  OpenAI-compatible endpoint serving both, Auto could land on the embedder. It now
  recognizes those names while still leaving real chat and coder models alone.

#### Agentic loop
- **Tool calls weren't repairable on native-tool models.** When a model emitted a
  tool call missing a required argument, only non-native models got an actionable
  "fix the parameters and try again" nudge — native-tool models (including local
  models served with tool calling) got just the raw error and would repeat or
  stall until a circuit breaker stepped in. Now every model gets a nudge that names
  the missing parameter with a per-tool hint.
- **In-loop verification always failed on Windows.** The auto-verify step appended
  a Unix-only no-op that isn't a command on Windows, so every verification reported
  a failure there. It now uses a cross-platform no-op.

### Added

#### Models & providers
- **Force native tool calling per model.** A new per-model control (Advanced
  Sampling panel → "Tool calling": Auto / Force on / Force off) makes Bodega send
  native OpenAI tool definitions to a model it didn't auto-detect as tool-capable —
  for a capable model behind vLLM or llama-swap exposed under a custom name.
- **MCP file-tool overlap warning.** When an enabled MCP server exposes file tools
  that overlap Bodega's built-in file tool, MCP settings now flag it — two ways to
  read and write files can confuse smaller local models. Detection only; nothing is
  disabled automatically.

#### Privacy
- **Turn off auto memory-extraction.** A new `memory.auto_extract` setting (on by
  default) stops the agent from mining facts out of conversations into the
  persistent memory store.

---

## [1.0.0-beta.31.3] - 2026-06-24

Verification, quality, and agent-control work, plus per-model sampling controls
and a round of fixes from live testing. Most of the verification pieces hang off
the existing QEL verification callback (`onQelTrace`), so they extend the
`qel_trace` event rather than adding new protocol surface.

### Added

#### Verification & quality
- **Post-loop code review.** After a creation task passes QEL, an optional
  quality pass reviews the files the agent actually changed (SRP violations,
  naming, obvious bugs) and surfaces the findings as a collapsible **Code
  Quality** section inside the existing verification card. It is non-blocking —
  it never gates completion — caps at five findings, skips files over 50 KB,
  bails to an empty result after 5 seconds, and sanitizes every finding before it
  lands in the trace. Off by default; turn it on with `qel.code_review`, and
  point it at a specific reviewer model with `qel.code_review_model` (empty = use
  the generating model). Creation tasks only. The finding shape rides the
  existing `qel_trace` event as an optional `codeReview` field — no new SSE event.
- **Rubric outcome grader.** Attach a free-text quality rubric to a task and,
  after QEL passes, a one-shot grader evaluates the output against it and pins a
  `{ verdict, score, justification }` result to the verification card. It is
  opt-in per request (the rubric rides the request body), runs on a dedicated
  small grader model when one is configured (`qel.grader_model`, falling back to
  the generating model), and short-circuits to *inconclusive* under air-gap, on
  an empty rubric, or on any grader error — it never breaks the stream. The
  result rides `qel_trace` as an optional `rubricResult` field — no new SSE event.

#### Agent control
- **Smart approval (intent-steered auto-allow).** In Ask mode, when the agent
  calls a low-risk read-only tool that clearly matches what you just asked for, it
  can auto-approve the call instead of interrupting you with an approval prompt.
  Write tools (`shell`, `file_system`, `str_replace`) are hard-blocked and always
  require a human click. The classifier runs *last* inside the approval handler —
  after every existing security gate — so it can only shrink the human-approval
  surface, never widen one. This first version is a static, no-LLM classifier:
  it auto-passes a known read-only set (`grep`, `glob`, `web_search`, `web_fetch`,
  `query_knowledge`, `query_memory`) at high confidence and scores intent/tool
  pairing against your last message for the rest. Off by default; enable with
  `permission.smart_approval` and tune the bar with
  `permission.smart_approval_confidence_threshold` (default 0.80). Both global.
  The `tool_approval` event gains two optional fields
  (`smartApprovalAttempted`, `smartApprovalReasoning`) — no new SSE event.
- **`/learn` — author a skill from a source.** A new `learn_skill` tool reads a
  source (a workspace directory via the sandboxed file-system tool, or a URL via
  the SSRF-protected web-fetch tool), runs one constrained authoring turn to draft
  a spec-conformant skill YAML, and validates it with the existing frontmatter
  parser. The draft is shown as a **preview**, then — on approval — written via
  the existing `POST /skills` route + registry reload. The write is
  permission-gated (Ask/Plan return 403 unless the request is explicitly approved;
  Act writes directly), and a `/learn` message is intercepted before
  trigger-matching so a user skill named "learn" can't shadow the command. URL
  sources are blocked under air-gap (directory sources still work locally); the
  source is truncated to ~10 KB before authoring, a surrounding markdown fence is
  stripped before validation, and malformed YAML returns an error rather than a write.
- **Auto-skill capture.** After a creation task passes QEL cleanly with enough
  tool calls to constitute a real workflow (≥5 calls, ≤1 repair iteration), the
  run can be captured as a lightweight, user-scoped knowledge card so a similar
  future request is auto-recalled at session start. Tool output is sanitized
  before it lands in the card, persistence is `user_id`-scoped, and it is off by
  default (`skills.auto_capture`). Local-only — safe under air-gap.

#### Models & sampling
- **Per-model advanced sampling.** Each installed model now has an Advanced
  Sampling panel, opened from the new Settings control on the model row, for
  per-model `top_p`, `top_k`, `min_p`, `repeat_penalty`, `repeat_last_n`, `seed`,
  and `stop` overrides alongside the existing temperature, max-tokens, and
  context controls. Samplers a given backend can't accept are greyed out per
  provider (`top_k` and `min_p` are local-only; `repeat_penalty` and
  `repeat_last_n` apply to Ollama), so you can't set a value the server will
  reject. Overrides are passed through to Ollama (inside `options`) and to local
  OpenAI-compatible servers including llama.cpp; cloud providers receive only the
  samplers they support.

### Changed

#### Agentic loop
- **Proactive auto context-compaction.** The agentic loop now compacts the
  conversation at a configurable fill threshold at iteration boundaries, instead
  of waiting for the hardcoded 85% emergency point or a manual `/compact`. A long
  run trims itself before it hits the wall. Set the threshold with
  `llm.auto_compact_threshold` (default 75; `0` disables proactive compaction and
  leaves only the 85% emergency backstop; valid values are `0` or 50–95). The
  compaction machinery is unchanged — the summary is still pinned as the second
  system message for KV-cache stability, and it still falls back to mechanical
  trimming if the summary call fails.

#### Fleet & diffs
- **Per-file churn badges in the parallel-run diff.** Each changed file in a
  parallel-run diff column now shows `+insertions` (green) / `-deletions` (red),
  so a reviewer can see at a glance which files carry the weight of a change.
  Binary files show a `binary` marker instead of counts. The parallel-run status
  feed now also carries the real per-session churn aggregate (it was hardcoded to
  zero) — cached per session and recomputed only when the diff changes, so the
  2-second status poll adds no git overhead. No new SSE event.

#### Benchmarking & headless
- **Aider-Polyglot benchmark prerequisites.** Groundwork for publishing an honest
  harness-lift number: the `POST /sessions/:id/abort` route now also aborts a
  headless run (so a benchmark harness can cancel a hung task and free the
  concurrency slot), `HeadlessSessionRunner` honors an optional `maxDurationMs`
  cap (a task can't wedge a batch), and a new adapter spec
  (`source-of-truth/specs/benchmark-aider-adapter.md`) documents the run procedure
  — fresh session per task, hidden tests as the oracle, harness-vs-raw-model LIFT.

### Fixed

#### Onboarding
- **Junk emails reaching the beta list.** The first-run beta gate requires an
  email, so a share of entries were deliberate garbage (random local part,
  made-up TLD) typed to get past it. The first-run screen and the backend now
  both check the address against a shared validator before it is accepted or sent
  onward: the TLD must be a real one (a two-letter country code or a recognized
  general TLD), and a short list of disposable providers is refused. Local
  activation still succeeds either way — the check only keeps the contact list
  clean. A plausible-but-fake address still passes; only a confirmation email
  would catch that.

#### Verification
- **QEL failed correct code over absent optional patterns.** A correct creation
  (for example, a Python module with tests) could score a failing 68/100 because
  optional language-idiom patterns that happened to be absent still counted
  against the score, and the Python compile/import proof only ran for server or
  app entry points. Optional language patterns are now bonus-only — their absence
  can no longer drag the score down — and the Python proof falls back to any
  `.py` deliverable. The same file now passes.

#### Settings
- **Wrong temperature-schedule description.** The Agent settings note described
  the dynamic temperature schedule backwards. It now matches what the loop
  actually does: steadier during planning, slightly higher during file writes.

---

## [1.0.0-beta.31.2] - 2026-06-23

The post-beta.31.1 marathon — a hands-off quality-of-life and provider-audit
campaign (a 172-finding audit worked high-to-low), the bugs Joe surfaced from live
diagnostics, and the preview / dev-server follow-ups.

### Fixed

#### Agentic loop & verification
- **Capable models hit the iteration limit mid-build.** A cloud-Opus build of a full
  multi-module game stopped unfinished at iteration 24. The effective budget was
  clamped by the model profile's recommendation (xlarge was only 20), so raising the
  user ceiling alone did nothing. Bumped the size-class recommendations (xlarge 20→35,
  large 15→20, medium 10→12) and the default ceiling (25→50) with a migration; the
  no-progress / degeneration circuit breakers still bound runaway loops.
- **Project builds stranded by the boilerplate firewall.** When the contract
  extractor mis-read a prompt (e.g. parsing "Three.js" as the lone deliverable from
  a "build a 3D game" request), ModeFirewall blocked `package.json`, `tsconfig.json`,
  and `vite.config.ts` as "not in the contract" — so the model made calls but wrote
  no files and looked stuck. Load-bearing web/JS scaffold is now exempt when the
  contract looks like a web/JS project; other ecosystems keep blocking superfluous boilerplate.
- **Contract extractor mined prose into a garbage contract** (the root cause of the
  above). A "build a 3D game with Three.js, steering (A/D)" prompt produced a lone
  deliverable `Three.js`, fake routes (`/D`, `/right`, `/outrun` from inline slashes),
  and fake functions (`steering` from "steering (A/D)"). Now: ~40 known library
  filenames (`three.js`, `react.js`, `d3.js`, `chart.js`, …) are never treated as
  deliverables; a route's leading `/` must not follow an alphanumeric (so "A/D" prose
  isn't a route); and a function call must have no space before `(` (so "steering (A/D)"
  prose isn't a function). Real filenames, `GET /api/users`, and `drawCar(ctx)` still extract.
- **Force-completed runs over-claimed success.** On an escape-hatch / force-complete
  exit, the summary read "Task complete: 1 file created" even when the QEL check had
  failed — directly contradicting the "⚠️ Quality check scored X/100" warning appended
  right below it. The lede is now an honest "Wrote N file(s):" (or "Finished — no files
  were written."), leaving the success/failure verdict to the QEL line.
- **Chat: duplicate turn on a truncated-stream retry + dropped queued sends.** A
  mid-stream retry could re-post the user turn, and follow-ups queued during a run
  could be lost on completion. Both fixed.

#### Providers
- **BYOK per-message cost showed $0** for Fireworks (`accounts/fireworks/models/<slug>`
  never matched) and cold-cache OpenRouter, plus longest-prefix mis-pricing. Pure
  id-resolution, no price guessing.
- **Cloud rate-limit over-throttling.** The pre-flight budget paced every cloud
  request against a conservative hardcoded default (Anthropic 30K ITPM), adding
  15–45s waits per iteration for higher-tier accounts. It now learns the real
  per-window limit from the provider's own response headers (override > observed > default).
- **Rate-limit buckets were shared across all OpenAI-compatible providers** (one 429
  on Groq throttled Together/OpenRouter/DeepSeek/Fireworks too) — now keyed per preset.
- **OpenAI `complete()` parity** with `stream()`: records the actual returned model,
  logs non-OK responses, fast-fails an unreachable host, honors `Retry-After`.
- **Anthropic**: broadened context-overflow detection for newer/1M-context errors;
  `complete()` connect-timeout + per-model `max_tokens` + a free (`count_tokens`)
  health check; extended-thinking blocks now round-trip so reasoning + tool-use no
  longer 400s. (`AnthropicProvider.ts` split under the 700-line cap to land these.)
- **Ollama**: tool-result linkage (preserved `tool_call_id` + derived `tool_name`),
  faster health recovery (3s fail-cache), broken-tool-template poison handling in `complete()`.
- **OpenRouter** one-sided pricing no longer drops the model; **Claude display names**
  keep their variant qualifiers ("Opus 4.8 Fast" vs "Opus 4.8"); **Fireworks**
  serverless slugs resolve to their dotted profiles; **Featherless** warmup persistence
  is idempotent; cloud-provider list completed (cohere/fireworks/qwen/kimi) + Groq onboarding.

#### Preview
- **Preview opened the wrong port.** Vite increments from 5173 (and users set custom
  ports like 5180), but both detectors only knew 5173/5174 — so the live server was
  invisible and a stale port showed. Both the terminal-output detector and the backend
  port scan now cover the Vite dev/preview increment band.
- **Preview opened the external browser instead of the in-app panel, and "open
  preview" could hang the model.** Two fixes (Joe 2026-06-23): (1) dev servers run
  with `BROWSER=none`, so a Vite/CRA `server.open: true` no longer pops the system
  browser — Bodega's in-app Preview owns it. (2) A foreground dev-server command
  (`npm run dev`, `vite`, `next dev`, …) is now auto-detected and run in the
  background instead of blocking until the exec timeout (which had made weaker models
  repeat themselves into the doom-loop guard). When the server prints its localhost
  URL, the in-app Preview opens to it automatically — no second `open_preview` step.

#### Security & multi-user
- **Sandbox escape via a symlinked parent directory** closed (realpath-walk to the
  nearest existing ancestor).
- **Per-user scoping** for memory routes and loop concurrency; PreToolUse hooks +
  permission profiles now also gate read-only tools (a `**/.env` deny stopped a read).

#### Code mode, settings, loops, fleet, onboarding
- Code-mode silent save/stage failures now surface; Command Sandbox dropdown persists;
  loop scheduler no longer resets unrelated loops' timers; fleet parallel runs
  recognize the backend `complete` status; settings import validation + reset-all
  air-gap sync; first-run onboarding fixes + a Reset-All confirmation dialog.
- **QEL/Drift verification card stayed pinned across sessions, with no way to
  dismiss it.** The inline card in the agent panel only cleared on the next send,
  so it lingered after switching to or creating a new session. Switching sessions
  now clears the trace, and the card has a × to dismiss the block (the Drift strip
  goes with it); it reappears on the next verification run. The dismiss is view-only
  — the Debug → QEL tab keeps the data.

### Added
- **`open_preview` tool** — one discoverable action that auto-detects the running dev
  server's port and opens the Preview panel (reusing the existing preview relay; no
  command execution).
- **`shell` `run_in_background`** — start a long-lived process (dev server, watcher)
  without blocking the call; returns the pid + early output, keeps running until the
  session is deleted or the app exits, tree-killed on cleanup. Same approval +
  blocked-pattern + credential-scan guarantees as the foreground path. (Pairs with `open_preview`.)
- **Cloud Boost default-model dropdown** — pick from the provider's curated/live model
  list instead of typing a model id.

### Changed / Docs
- **TOOLS.md** regenerated against the registry (documents `preview_interaction`,
  `open_preview`, `vision_query`; 27 tools).
- **Design spec** for serializing llama.cpp stop/swap (one lifecycle lock; closes the
  #90/#93 Windows file-lock races) — `source-of-truth/specs/`, for review before build.

---

## [v1.0.0-beta.31.1] - 2026-06-22

A hotfix for the OpenRouter and Fireworks providers and how the app handles
cloud models that cold-start on the first request.

### Fixed
- **OpenRouter "Cannot reach OpenRouter" banner.** A single slow health poll
  (OpenRouter's large model catalog occasionally taking longer than its fetch
  window) would flip the app to "disconnected" and blank the model picker. The
  health check now needs two consecutive failed polls before showing the banner,
  one healthy poll clears it instantly, and the cloud model-list fetch gets a
  longer timeout so the catalog finishes and caches instead of re-failing.
- **OpenRouter "Request timed out" on a cold model.** OpenRouter returns headers
  immediately, then holds the stream open while it warms up the upstream model.
  If that ran long it sent an error before any token, and the app was silently
  swallowing it and ending with an empty reply. Pre-token stream errors now
  surface, cloud providers automatically retry the transient ones (safe, since
  nothing was streamed yet and the first request already warmed the model), and
  the cloud connect window was widened so a slow-but-healthy start is not cut off.
- **Fireworks empty model picker.** Fireworks' model-list endpoint errors for
  serverless accounts, so the picker came up blank and the provider looked
  broken. Bodega now ships a curated list of current Fireworks serverless models
  (DeepSeek V4 Pro/Flash, GLM 5.2, Qwen3.7 Plus, Kimi K2.5, MiniMax M3, Nemotron
  3 Ultra, GPT-OSS 120B/20B), each verified to chat, stream, and tool-call. The
  key-validation probe was refreshed to a current model as well.

### Added
- **OpenRouter integration polish.** Accurate per-message cost tracking from
  OpenRouter's live pricing; the model picker organized by vendor and popularity
  (most-used first) with search across the full catalog; previously hidden Llama
  and Gemini models restored to the list; and app-attribution headers so Bodega
  One registers on OpenRouter's app directory.

---

## [v1.0.0-beta.31] - 2026-06-22

The advanced-automation and verification wave. Named permission profiles and
lifecycle hooks gate what the agent is allowed to do; regression contracts plus
the new Drift Radar re-prove work you've already verified; and QEL verification
now handles vanilla JavaScript and several more languages correctly. Also lands
contract preview in the composer, MCP per-server tool filtering, path-scoped
project rules, a configurable shell environment policy, custom llama.cpp
arguments, opt-in OpenTelemetry export, air-gap attestation, and proof-carrying
commits.

### Added
- **Lifecycle hooks (PreToolUse / PostToolUse).** You can now run your own shell
  command before or after the agent uses a tool — to lint, format, validate, or
  **block** a change before it happens. A `PreToolUse` hook can stop a tool (exit 2)
  or rewrite its input; a `PostToolUse` hook observes the result. Hooks you write in
  your own settings are trusted; hooks committed in a repo's `.bodega/config.json`
  are **never run until you approve them** (per exact command + project — a cloned
  repo can't run a hook behind your back). HTTP hooks are deferred for now; shell
  hooks cover the common lint/format/gate case. Off when you have none configured.
- **Named permission profiles.** Define reusable named rule sets that **narrow** what
  the agent may do — deny a tool by name, or allow/deny file writes by path glob
  (e.g. a `readonly` profile, or one that confines edits to `src/**` and blocks
  `src/secrets/**`). A profile can only *restrict* access (never widens it) and
  composes with Ask/Plan/Act. Off until a profile is activated.
- **Regression contracts (the oracle store).** When a creation task passes QEL
  verification, Bodega now remembers the verified contract (deliverables + framework
  + the score it passed at) for that project — a durable "this worked" baseline. It
  de-duplicates automatically and is the foundation for upcoming drift/regression
  detection.
- **Drift Radar.** Bodega can now re-prove the contracts it has already verified for a
  project against your current code — telling you which previously-working deliverables
  still hold, which **regressed**, which had their proof break, and which simply
  **changed** (a retire candidate). Run it on demand from the new **Drift** tab in the
  code-mode debug panel, or let an optional nightly sweep watch in the background and
  raise a top-bar pill plus an in-app/OS notification the moment something newly
  regresses (reusing your existing Fleet notification settings). It is **report-only** —
  it never edits, applies, or auto-retires anything — and does no network or process
  spawning, so it's air-gap-safe. The background sweep is **off by default** (enable it
  under Settings, default nightly at 03:00).
- **Custom llama.cpp arguments.** Power users can now pass raw `llama-server` flags that
  aren't in the GUI — to tune for their exact hardware (e.g. `--n-cpu-moe`,
  `--override-tensor`, batch sizes, `--no-mmap`). Set them globally under Settings →
  Models → Advanced llama.cpp flags, or per-model for the currently-loaded model; a live
  preview shows the resolved command. Custom args override the GUI fields (last-wins).
  Flags Bodega must own (model selection, the `127.0.0.1` binding) are ignored with a
  notice, and `--rpc` is blocked under air-gap. The setting is global-only — a project's
  committed config can never inject server flags.
- **Watch-mode comment triggers.** A new Loop trigger kind watches a project folder
  and acts on inline comments when you save: `// bodega: <instruction>` runs that
  instruction on the file, and `// fix this AI!` asks the agent to fix the nearby
  issue. The change runs through the same QEL-verified, worktree-isolated apply gate
  as every other Loop (parked for review by default), and the trigger comment is
  removed after a verified change. **Off by default** behind two switches (the Loops
  master toggle + a dedicated *Watch for comment triggers* toggle in Settings →
  Automation); guarded against re-fire loops and double-dispatch on rapid saves.
- **OpenTelemetry audit export (opt-in, loopback-only).** Bodega can now export its
  existing agent-run telemetry (LLM calls, tool calls, proof gates, violations,
  run completions) as OpenTelemetry (OTLP) trace spans to an OTLP collector you run
  on `localhost` — for audit, compliance, or just seeing a run as a waterfall in
  Jaeger/Grafana Tempo. It's **off by default** and **loopback-only**: the endpoint
  is checked against a localhost allowlist before every send, so a non-loopback URL
  is refused (even with air-gap off). No new dependencies, and free-text fields are
  truncated before they leave the process. Enable it with `telemetry.otlp_export_enabled`
  + `telemetry.otlp_endpoint` (default `http://localhost:4318/v1/traces`).
- **Air-Gap Attestation.** Settings → Safety → Network Privacy now has a **Copy
  attestation** action that produces a signed, tamper-evident record of your
  air-gap posture: whether air-gap is on, how many outbound fetches were recorded,
  and how many blocked outbound attempts the enforcement layers counted since the
  app started — broken down by category (GitHub, git push, MCP, model downloads,
  etc.). It's HMAC-signed with a secret that never leaves your machine, so the
  record can't be edited after the fact. Honest by design: it attests *recorded
  fetches and counted refusals*, not a guarantee that zero bytes ever left the
  machine. Generating or verifying it makes no network calls.
- **Proof-Carrying Commits.** When you commit QEL-verified work, Bodega can attach
  a signed `Bodega-QEL` git trailer to the commit message — the score, pass/fail,
  execution-proof result, model, and a hash of the contract that was verified. The
  trailer is signed with an HMAC keyed to a secret that never leaves your machine,
  so a "verified" claim can't be forged or edited after the fact. A new **Verify
  history** action in the source-control panel walks recent commits and re-checks
  each trailer, marking them verified / tampered / unsigned. Tick **Add QEL proof**
  beside the commit button to include it. Fully local — building or checking a
  trailer makes no network calls.
- **Contract preview in the composer.** As you type a task, a small card shows what
  the agent will be held to before you send — the files it's expected to produce, the
  framework it detected, and any API routes it'll touch. It's the same execution
  contract QEL verifies the result against, surfaced up front so you can catch a
  misread ("it thinks this is a Flask app") before spending a run. Fully non-blocking
  and read-only: it never starts a run or persists anything, and it stays hidden for
  plain conversational messages.
- **MCP per-server tool filtering + required servers.** An MCP server config can now
  carry an `enabledTools` allowlist or a `disabledTools` denylist, so only the tools
  you choose from a server are exposed to the agent (instead of all-or-nothing per
  server). A server marked `required` fails fast — it isn't silently retried, and a
  failure to connect is surfaced — so a run can't quietly proceed without a tool
  source it depends on. Filtering sits below the air-gap/enabled security floor (it
  can only ever remove tools, never re-admit a blocked one).
- **Path-scoped project rules.** A project-rules file (`.bodega-rules` / `CLAUDE.md` /
  `.clinerules` / `.cursorrules`) can now carry rule fragments that apply only when
  the agent works on matching files — fence them with
  `<!-- bodega:rules paths="app/**,**/*.sql" -->` … `<!-- /bodega:rules -->`. The
  matching fragments are injected alongside your always-on global rules, chosen from
  the files the turn touches (open editor tabs + the files you name). Rules files with
  no fenced blocks behave exactly as before. Cuts wasted context and raises rule
  relevance.
- **Configurable shell environment policy.** A new `shell.env_policy` setting
  controls which environment variables reach shell commands the agent runs:
  `core` (default — the existing curated allowlist, unchanged), `none` (bare shell
  essentials only), or `full` (pass everything through). `shell.env_allow_extra`
  lets you name specific extra variables (e.g. a build token) to pass under the
  stricter policies. The default is byte-identical to today; `full` is still
  scanned for leaked credentials in command output. The policy is a global setting
  on purpose — a committed project config can't widen it.

### Changed
- **Forced routing tier resolves through the backend.** When you pin a routing tier
  with the Fast / Smart / Code pill, the model is now resolved by the backend's
  router (its real slot config) instead of the renderer's settings mirror — so the
  model shown in the status bar is exactly what the backend will serve. Matches the
  new `bodega run --tier` / `/tier` in the CLI. Auto routing is unchanged; the
  consult is short-timeout with a local fallback, so it never blocks a send.

### Fixed
- **QEL verifies vanilla JavaScript correctly.** Plain `.js`/`.jsx` deliverables (no
  TypeScript, no tsconfig) were being checked with `npx tsc`, which fails instantly in
  a dependency-free project and sank otherwise-correct creations to a failing score.
  JavaScript now gets a parse-only `node --check` per file (matching the existing Ruby
  and PHP gates), and language-pattern plus proof-gate coverage was widened to
  `.mjs`/`.cjs`/`.php`/`.c`/`.cpp`/`.swift` so valid code in those languages isn't
  scored zero. A missing third-party import is treated as an environment gap rather
  than a code error, and a config-only run that writes no deliverables no longer earns
  a free completeness pass.
- **Stale regression contracts no longer hide new regressions.** After you retired and
  re-verified an oracle contract, Drift Radar could keep reporting it as regressed from
  before, masking a genuine new failure. Findings now carry a baseline timestamp and
  the transition state resets on retire/reactivate, so a re-verified oracle reports its
  next regression as expected.
- **QEL and Drift results show inline in the code-mode agent panel.** They previously
  appeared only in the Debug tab. The llama.cpp model name now renders correctly in the
  model-role picker, and embedding-index logging no longer floods the console when
  indexing is off or the provider isn't configured.
- **Release downloads never 404 mid-release.** There was a 5 to 18 minute window during
  a release where downloads returned 404: the GitHub release went public before all
  platform assets finished uploading. Releases now upload assets into a draft and only
  flip to public and latest once every platform completes.

---

## [v1.0.0-beta.30.1] - 2026-06-20

A reliability hotfix for local models: your per-model context setting actually
applies again, a full context window no longer crashes the chat, your per-model
thinking toggle finally takes effect, setting the context too high now warns you
instead of failing silently, and an agent that gets stuck repeating itself now
gets caught instead of spinning.

### Fixed
- **Per-model context size applies again.** Setting a context window on an
  individual llama.cpp model had no effect — every model silently fell back to a
  4K window, so longer conversations broke for no apparent reason. The per-model
  value (or, failing that, the model's own trained length) is now wired into how
  the local server starts.
- **A full context window no longer crashes the chat.** When a local llama.cpp
  model ran out of context mid-reply, the overflow was misread as a fatal error
  instead of a recoverable one, and the chat died. It's now recognized and
  handled gracefully so the conversation can recover.
- **Ollama context overflows are recognized too.** The same class of "out of
  context" error coming back from Ollama is now caught and handled as an overflow
  rather than surfacing as a generic failure.
- **The per-model thinking toggle works on local models.** For llama.cpp models
  that support thinking (Qwen3, DeepSeek-R1, QwQ), the reasoning control you set
  per model was saved but never applied — the model thought (or didn't) on its
  own regardless. The toggle now takes effect: turn it off to get fast, direct
  answers, or on to let the model reason. Coder and embedding variants are left
  alone, since forcing thinking on them errors.

### Added
- **Stuck agents get caught (doom-loop detection).** If the agent calls the exact
  same tool with the exact same arguments over and over — a dead-end loop that
  burns time without making progress — it's now nudged to change approach, and if
  it keeps going, the run is wrapped up and graded rather than spinning forever.
  Genuinely repeatable actions like re-running tests or shell commands are exempt,
  so real work is never cut off.
- **A clear warning when the context is set too high.** If you set a local model's
  context window beyond what your hardware can allocate, the model card now warns
  you before you load it, and if a load does fail for that reason you get a plain
  "context window too high — lower it in Settings" message instead of a cryptic
  crash.

---

## [v1.0.0-beta.30] - 2026-06-20

The agent gets a memory and a sharper conscience: it learns from past work, your
automations improve themselves, and the quality checks now fire on the runs that
used to slip through unverified.

### Added
- **Loops learn from their own history.** A scheduled Loop now reads its recent
  runs and carries forward what failed, so it stops repeating the same mistakes
  every cycle. A Loop whose quality is steadily dropping auto-pauses instead of
  grinding on. Loops can also pace themselves (backing off when a run changes
  nothing), expire on a date, cap their total runs, and be defined in a committed
  `.bodega/loop.md` file so a repo can ship its own automations.
- **The agent learns across tasks.** It records what worked on past similar
  tasks and surfaces those lessons on the next one, reflects after each run to
  distill a short lesson, and learns from the tools you reject. All of it stays
  on your device, scoped to you and the project, and honors air-gap. This is
  in-context memory, not model fine-tuning.

### Fixed
- **Quality checks now fire when they matter.** The Quality Enforcement Layer
  used to skip grading when a model thrashed and the loop force-completed, or
  when a request was phrased loosely ("make me a dashboard") instead of naming
  files. Those runs slipped through marked as passed. They are now graded
  honestly with a real score and a repair path, no matter how the run ended or
  how the request was worded. Verified live across a small local model, a large
  local model, and a cloud model.

### Changed
- **Learning data is scoped per user.** Groundwork for shared deployments: every
  learned rule, recorded mistake, cached tool pattern, and model-performance
  record is now tagged to its owner so nothing crosses between accounts.

---

## [v1.0.0-beta.29.6] - 2026-06-19

Hotfix for bugs found while battle-testing the agent against cloud providers:
multi-file edits could fail outright, and a denied tool could still run.

### Fixed
- **Multi-file tasks no longer fail with a 400 on cloud providers.** A turn that
  called several tools at once (e.g. editing multiple files) could error with a
  provider 400 on OpenAI-compatible / native-function-calling backends, because
  the assistant tool-call and tool-response messages were only reconciled one
  way. The backstop is now bidirectional, so multi-tool turns pair up correctly.
  Local / XML-tool providers were never affected.
- **Duplicate tool responses are no longer emitted** for a native-function-calling
  tool call — part of what tripped the provider 400.
- **A denied tool is now actually blocked.** A tool named in a deny list could
  still execute (act mode auto-approved it before the denial was checked). Denial
  is now enforced server-side, before the tool runs, in every permission mode.

### Added
- Regression tests for the bidirectional tool-pairing backstop and the
  server-side deny-list enforcement.

---

## [v1.0.0-beta.29.5] - 2026-06-18

Reliability and privacy pass on the beta.29 line: local models boot cleanly,
opening a file gets a real lightweight viewer, the editor works fully offline,
and beta sign-up no longer loses contacts.

### Added
- **Lightweight file viewer.** Opening a file with Bodega One (a file
  association or "Open with") now launches a fast, theme-consistent viewer
  window instead of booting the whole app and loading a model. Markdown shows a
  formatted preview; any file opens in a real Monaco editor with line numbers, a
  minimap, and syntax highlighting. You can edit, save with Ctrl/Cmd+S, and get
  an unsaved-changes prompt on close. "Open in Bodega One" promotes the file
  into the full app when you want it.

### Fixed
- **Local-model boot reliability (llama.cpp).** Local models now recover from a
  crashed llama-server instead of wedging, no longer fall back to Ollama once
  you have picked a local GGUF, and load in the background so the window stays
  responsive while weights load. Speculative-decoding (MTP) draft models resolve
  from the correct location, with a migration for installs that used the old
  path.
- **Editor works fully air-gapped.** The Code-mode editor now ships its Monaco
  bundle inside the app instead of fetching it from a CDN at runtime. With
  air-gap on, opening the editor makes no outbound requests. The editor still
  loads on demand, so the main bundle stays within its size budget.
- **Beta sign-up no longer drops contacts.** First-run email capture now writes
  every submission to a durable on-device queue before contacting Loops, then
  retries in the background. If you were offline or hit a network error at
  sign-up, your contact is held and sent later instead of being silently lost.
  Air-gap still sends nothing until you turn it off.

## [v1.0.0-beta.29.4] - 2026-06-17

Fixes a broken llama.cpp local-model experience reported on beta.29.3.

### Fixed
- **Downloaded models now show in My Models.** A GGUF you downloaded could go
  missing from My Models (and stay unloadable) whenever llama-server wasn't
  already running, because the list came from the live server instead of what's
  installed on disk. Installed GGUFs now always appear under the llama.cpp
  preset, so you can load one even before the server starts.
- **No more stuck "Unknown GGUF id" startup.** If a model id from a previous
  Ollama setup was left as the llama.cpp default, the server refused to start.
  Bodega now ignores and clears that stale id and stays idle until you pick a
  model, instead of failing the whole local-model setup.
- **No silent fall-back to Ollama.** With no model loaded, llama.cpp could quietly
  switch to Ollama and then log a connection error every 30 seconds forever (even
  when Ollama wasn't installed). It now stays on llama.cpp, idle.
- **Clearer "Test Connection".** Testing the llama.cpp connection with no model
  loaded now explains that a model must be loaded first, instead of a raw
  "fetch failed".

## [v1.0.0-beta.29.3] - 2026-06-17

Hotfixes on top of the beta.29 routing release. beta.29.1 and beta.29.2 shipped
as silent auto-updates; their changes are recorded here too.

### Fixed
- **Local-provider install progress.** Installing llama.cpp or Ollama from
  Settings → Providers now streams real download progress, keeps running when you
  leave the tab, and has a Cancel button. It no longer looks cancelled when you
  navigate away, and the llama.cpp card no longer offers to install a binary
  that is already present. (beta.29.2)
- **Model downloads survive a restart.** A GGUF download that is still running
  re-attaches its progress bar after a full app restart instead of showing an
  idle Download button. (beta.29.3)
- **"What's New" shows your version.** The release notes no longer render a stale
  "Unreleased" heading on a shipped build. (beta.29.3)

### Added
- **Refreshed model catalog.** Five current GGUF models join the llama.cpp
  catalog (Qwen3-Coder-30B, Gemma 4 12B, DeepSeek-R1-0528-Qwen3-8B, Qwen3.5-9B,
  and Qwen3-VL-30B), and the managed llama-server binary moves to b9670.
  (beta.29.2)
- **Name on the first-run beta gate.** Beta activation now captures your name
  alongside your email. (beta.29.1)

## [v1.0.0-beta.29] - 2026-06-15

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
