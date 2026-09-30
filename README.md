# `aicli` — Unified Local AI Developer Suite

`aicli` is a lightweight, fully offline AI command-line toolkit powered by [Ollama](https://ollama.com).
One binary covers four workflows: quick shell questions, system-design blueprinting, **autonomous
multi-file project generation**, and local model management — no cloud APIs, no accounts, no data
leaving your machine.

```
ai "How do I find files modified in the last 24 hours?"
aarch "Design a modular Python web scraper with a config parser and a database logger."
abuild "A counter CLI tool with config settings, a core logic library, and a main script."
```

---

## Table of contents

- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Command reference](#command-reference)
  - [`aicli ask` (alias `ai`)](#aicli-ask-alias-ai)
  - [`aicli arch` (alias `aarch`)](#aicli-arch-alias-aarch)
  - [`aicli build` (alias `abuild`)](#aicli-build-alias-abuild)
  - [`aicli agent` (alias `aagent`)](#aicli-agent-alias-aagent)
  - [`aicli model`](#aicli-model)
  - [`aicli config`](#aicli-config)
- [Configuration](#configuration)
- [How `build` works under the hood](#how-build-works-under-the-hood)
- [Security notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Project layout](#project-layout)
- [Contributing](#contributing)
- [Versioning](#versioning)
- [License](#license)

---

## How it works

`aicli` is a single Bash script (`aicli`) that shells out to your local Ollama daemon. The four
shortcuts — `ai`, `aarch`, `abuild`, `aagent` — are either symlinks to that script or tiny wrappers next to
it; either way they end up running the same code.

| Command | What it does |
|---|---|
| `aicli ask` / `ai` | Sends one prompt to the model and prints the answer. Your terminal Q&A. |
| `aicli arch` / `aarch` | Asks the model to act as a software architect: blueprint + JSON file tree for a project idea. |
| `aicli build` / `abuild` | Two-phase autonomous builder: first the model plans the project as a JSON file list, then `aicli` loops over that list, generating each file with the previously generated files fed back in as context. |
| `aicli agent` / `aagent` | Sandboxed multi-model agent: an architect model plans, a builder model works the filesystem with tools (`read`/`write`/`append`/`ls`/`run`/`done`), and a reviewer model critiques the result and sends it back for fixes. |
| `aicli model` | Lists downloaded Ollama models, shows the active one, switches it (pulling it first if needed). |
| `aicli config` | Shows/changes settings like the active model and the build context window. |

Everything runs through `ollama run <model>` on your own machine. Nothing is uploaded anywhere.

---

## Requirements

- **Bash** 4+ (macOS ships Bash 3.2 — install a newer Bash via Homebrew for full support)
- **[Ollama](https://ollama.com)** installed and running, with at least one model pulled (e.g. `ollama pull qwen2.5-coder:7b`)
- **python3** — used to validate and parse the architect's JSON plan
- **perl** — used to strip `<think>` blocks and markdown fences from model output

`aicli` checks for all of these on startup and tells you exactly what's missing.

---

## Installation

Clone the repo anywhere you like (the examples use `~/aicli`):

```bash
git clone https://github.com/ksujiththomas/ai-cli-toolkit.git ~/aicli
```

Then pick **one** of these two methods:

### Method A — symlinks (recommended)

Point all five commands at the one script:

```bash
mkdir -p ~/.local/bin
ln -sf ~/aicli/aicli ~/.local/bin/aicli
ln -sf ~/aicli/aicli ~/.local/bin/ai
ln -sf ~/aicli/aicli ~/.local/bin/aarch
ln -sf ~/aicli/aicli ~/.local/bin/abuild
ln -sf ~/aicli/aicli ~/.local/bin/aagent
```

Make sure `~/.local/bin` is on your `PATH` (add this to `~/.bashrc` or `~/.zshrc` if it isn't):

```bash
export PATH="$HOME/.local/bin:$PATH"
```

This works because `aicli` detects the name it was invoked as: running it via the `ai`
symlink is equivalent to `aicli ask`, `aarch` to `aicli arch`, `abuild` to `aicli build`,
and `aagent` to `aicli agent`.

### Method B — repo directory on PATH

```bash
export PATH="$HOME/aicli:$PATH"
```

Here the `ai` / `aarch` / `abuild` / `aagent` wrapper scripts in the repo do the dispatching —
each one finds the real `aicli` script sitting next to it (symlink-safe) and calls it
with the right subcommand. Use this if you'd rather not create symlinks.

Verify the install:

```bash
aicli --version
ai "Reply with the word OK."
```

---

## Quick start

```bash
# 1. See which model is active and what's downloaded
aicli model

# 2. Ask a quick question (alias: ai)
ai "How do I list all listening TCP ports with ss?"

# 3. Get an architecture blueprint (alias: aarch)
aarch "Design a REST API for a todo app with SQLite storage."

# 4. Build a whole project from one sentence (alias: abuild)
mkdir my-counter && cd my-counter
abuild "A counter CLI tool: config/settings.conf, lib/counter.sh, and bin/counter"
ls -R
```

---

## Command reference

### `aicli ask` (alias `ai`)

Your quick shell assistant. Sends a single prompt to the active model and prints the reply.

```bash
aicli ask "How do I find all files modified in the last 24 hours?"
ai "Explain what 'set -euo pipefail' does in bash."
ai -a "short flag form also works"
```

Multi-word prompts don't need quoting tricks — everything after the subcommand is joined
into one prompt (`"$*"`), so `ai what is 2+2` works too. Quotes are still recommended when
your prompt contains shell special characters.

### `aicli arch` (alias `aarch`)

Architecture mode. The model is instructed to respond as a principal software architect with
a high-level blueprint, component design, and a JSON file tree for your project idea. Use it
to sanity-check a design *before* you build.

```bash
aicli arch "Design a modular Python web scraper with a config parser and a database logger."
aarch "Microservice layout for an image-resizing pipeline on a single VPS."
```

Tip: pipe it somewhere for later — `aarch "..." > docs/architecture.md`.

### `aicli build` (alias `abuild`)

The autonomous builder. Give it one sentence describing a project; it plans the file layout
and writes every file for you. See [How `build` works under the hood](#how-build-works-under-the-hood)
for the full pipeline.

```bash
mkdir netmon && cd netmon
abuild "A 2-file Bash tool: config/net.conf and scripts/netmon.sh"
```

**What you get:**

- Each planned file is created at its relative path; parent directories are made automatically.
- Files ending in `.sh`, files under `bin/`, or files starting with a `#!` shebang line are
  marked executable.
- Generation is **cumulative**: every file already written is included as context when the
  next file is generated, so imports, config keys, and relative paths stay consistent.
- Unsafe paths are rejected: if the model ever emits an absolute path or a `..` segment,
  the build aborts instead of writing outside your project directory.

Example session:

```
$ abuild "A counter CLI: config/settings.conf, lib/counter.sh, bin/counter"
===================================================
 PHASE 1: Architecting System & File Tree (Model: qwen2.5-coder:7b)
===================================================

Architect generated a 3-file plan.

===================================================
 PHASE 2: Autonomous Code Building Loop (Model: qwen2.5-coder:7b)
===================================================

---------------------------------------------------
Building [1/3]: config/settings.conf
Purpose: Key/value settings: initial count, step size, state file path.
---------------------------------------------------
Saved -> config/settings.conf (8 lines)
...
===================================================
 SUCCESS: Project build complete!
===================================================
```

**Always review generated code before running it** — especially anything with `sudo`, `rm`,
or network calls. The model is a fast drafter, not a reviewer.

### `aicli agent` (alias `aagent`)

A sandboxed, multi-model autonomous agent for tasks that need the model to *do things* —
not just generate text, but read, write, and run code in your project directory, then verify
its own work.

It runs in three phases, each of which can use a **different local model**:

1. **Architect** (`ARCHITECT_MODEL`) writes a short step-by-step plan **plus a
   machine-readable deliverables list** — every file the task requires.
2. **Builder** (`BUILDER_MODEL`) works the plan as a tool-calling loop: it emits one tool
   call per turn — `read`, `write`, `ls`, `run`, `done` — and `aicli` executes it and feeds
   the result back, up to `AGENT_MAX_STEPS` turns. Every prompt it sees leads with a live
   `[done]` / `[MISSING]` checklist of the deliverables, so it can't lose track of what
   still needs creating.
3. **Reviewer** (`REVIEWER_MODEL`) reads every file the builder changed, checks them
   against the deliverables list, then either approves or sends file-specific revision
   notes back to the builder (up to `AGENT_MAX_REVISIONS` rounds).

```bash
aagent "Add a --verbose flag to bin/app and test it"

# Per-run overrides:
aagent --mode confirm "Refactor config parsing into config/lib.sh"
aagent --architect-model llama3.1:8b --builder-model qwen2.5-coder:7b \
       --reviewer-model llama3.1:8b --no-review "Write a Makefile with test and clean targets"
```

**Autonomy modes** (`--mode`, or `aicli config agent-mode <mode>`):

| Mode | What happens when the agent wants to run a shell command |
|---|---|
| `sandbox` (default) | Runs inside a [bubblewrap](https://github.com/containers/bubblewrap) jail: the project dir is read-write, the rest of the system is read-only, no network, no home directory. If `bwrap` isn't installed, falls back to `confirm` with a warning. |
| `confirm` | Prints the command and asks `Allow this command? (y/N)` every time. |
| `dry-run` | Prints every tool call but executes nothing — a safe rehearsal. |

Safety is layered, not optional: every path is jailed to the directory you run in
(absolute paths and `..` are rejected), destructive/privileged commands (`sudo`, `mkfs`,
`dd`, `rm -rf /`, …) are denied by policy even inside the sandbox, each command is
time-limited (`AGENT_CMD_TIMEOUT`), and tool outputs are truncated (`AGENT_MAX_OUTPUT_CHARS`)
so a runaway `cat` can't blow the model's context window.

**How the builder stays on track.** Small local models wander: they rewrite one file
forever, chatter without calling tools, or declare victory on a skeleton. The harness
compensates, so you don't have to babysit:

- **Deliverables checklist** — every builder prompt starts with the architect's file list
  marked `[done]` / `[MISSING]`, so the model always knows what's left.
- **`done` is gated** — calling `done` while deliverables are still missing is rejected
  with the missing list; three rejections in a row aborts the run.
- **Rolling history** — only the last few tool exchanges are kept verbatim
  (`AGENT_HISTORY_KEPT`, default 6); the checklist carries the state, so long runs
  don't drown the model in context.
- **Stuck detectors** — rewriting the same file with identical content three times
  triggers a redirect to the missing files; ignoring the tool protocol three times in
  a row aborts with a suggestion to use a larger model.
- **Forgiving parser** — slightly malformed tool calls (JSON wrapped in prose) are
  recovered automatically instead of failing the step.

#### Project mode: `aagent --project`

For multi-file builds, `--project` switches the agent from file-at-a-time to
**section-by-section** construction:

```bash
aagent --project "Build a blog with index, archive, and post pages plus shared CSS"
```

The architect breaks every deliverable into 2–5 ordered sections, and the harness writes
them to `.aicli/progress.md` — a checklist the builder works top to bottom. Two new
tools make the loop work:

- `append` — adds a section to an existing file (refused if the file doesn't exist yet,
  so the model can't append onto nothing). Also available in `--chat` and plain
  one-shot mode.
- `progress` — checks a section off in `.aicli/progress.md`. The file is
  harness-owned: the model signals via the tool, and the harness flips the box, so a
  weak model can't corrupt the format or fake completion.

The builder's discipline per section: `read` the file's current content, `write` (first
section) or `append` (later sections), then `progress` to check it off. `done` is gated
on **zero unchecked boxes** — an early `done` is rejected with the remaining sections
listed. The reviewer also sees the section checklist. Default `--max-steps` rises to 40
in project mode (overridable).

Because the checklist lives on disk rather than in context, the model can't lose its
place on long builds — this is the mechanism that lets a 14B model reliably finish
work that used to stall, loop, or end in placeholders.

#### Manual plan: `aagent --project --plan-file <file>`

When the architect is the bottleneck -- a small local model that plans too slowly
or unreliably for a complex task -- skip it and hand the builder your own checklist:

```bash
aagent --project --plan-file /tmp/phase-basic-plan.md
```

The file uses the same format as `.aicli/progress.md`: top-level
`- [ ] <path> -- <purpose>` lines become the sections (and populate the
deliverables list behind the `done` gate), with `  - [ ]` sub-items for the
checkable details:

```markdown
# Build progress
- [ ] index.html -- homepage
  - [ ] head: meta charset, viewport, title, stylesheet link
  - [ ] hero: video autoplay muted loop playsinline src assets/hero-video.mp4
- [ ] styles.css -- site styling
  - [ ] :root variables, box-sizing reset, body font and colors
```

The harness copies your file to `.aicli/progress.md` and the builder works it top
to bottom exactly like an architect-produced plan. The plan critique loop is also
skipped -- your checklist is used as written. Reach for this when you know exactly
what you want built, or when the architect model is too slow for the job.

#### Thorough mode: `aagent --project --thorough`

When quality matters more than speed, `--thorough` gives both the architect and the
builder more power — expect a run to take several times longer:

```bash
aagent --project --thorough "Build a blog with index, archive, and post pages plus shared CSS"
```

- **Architect plans against reality.** The architect prompt now always includes a
  project snapshot (files, 2 levels deep), so the plan accounts for what already
  exists instead of planning blind.
- **Plan critique loop.** After the architect drafts the plan, a critic
  (the reviewer model, or the architect itself under `--no-review`) reviews it for
  missing files, oversized sections, and wrong build order. `revise` verdicts send
  the architect back with concrete notes; `approve` starts the build. Up to 3 rounds.
- **Builder verifies per section.** The builder is instructed to re-read each file
  after every `write`/`append` and only check off sections that are truly done,
  fixing problems before moving on.
- **Higher caps.** `--max-steps` defaults to 80 and `--max-revisions` to 5 under
  `--thorough` (both still overridable).

**Visibility.** Every run now mirrors its full output to
`.aicli/run-<timestamp>-<pid>.log`, so you can `tail -f` progress from another
terminal or review a finished run. Step lines show live checklist progress
(`--- step 12/80 [sections 4/9] ---`), per-step durations, and a `loading ... (first
load from disk can take a minute)` note before the first model call so a cold start
doesn't look hung.

**Hardening.** Two circuit breakers cover the ways a weak model actually dies:

- **Architect retry.** If the architect returns no parseable `<DELIVERABLES>` block,
  the harness retries up to 3 times (with a stricter format reminder), instead of
  silently disabling the done-gate and letting the builder flail without a plan. Raw
  architect output is saved to `.aicli/architect-attempt-N.raw` for forensics.
- **Repeat-call breaker.** The same tool with the same arguments 3× in a row (e.g.
  17× `ls .`) gets a `LOOP WARNING` redirect instead of executing; 6× aborts the run.
  `done` and `write`/`append` are exempt — rejected dones and identical rewrites have
  their own dedicated detectors.

**Timing report.** Each run ends with `AGENT timing:` — per-phase durations
(architect with critique-round count, builder with step count, reviewer), per-section
durations slowest-first, the 3 slowest steps, and a `slowest phase: X (N%)` line so
you can see where the time actually goes and what to optimize next.

**Model size matters more than anything else here.** A 7B coder model can run the agent
loop, but it will need the guardrails above to finish real tasks. For noticeably better
agency, give the architect and reviewer a larger reasoning model and keep a fast coder
model on builder duty:

```bash
aicli config architect-model qwen2.5-coder:32b
aicli config builder-model qwen2.5-coder:14b
aicli config reviewer-model qwen2.5-coder:32b
```

(Each role falls back to `CHAT_MODEL` when unset, so this is opt-in.)

#### Interactive chat: `aagent --chat`

For back-and-forth work, `--chat` turns the agent into a conversational assistant with
tools. Instead of one task in / one result out, you get a REPL: you talk, it acts with
`read` / `write` / `ls` / `run`, narrates what it's doing, and waits for your next
instruction. A reply with no tool calls is its message to you — so it can also ask
clarifying questions instead of guessing.

```bash
cd ~/my-site
aagent --chat
# > You: add a dark mode toggle to the nav
# Muse: I'll add a toggle button and wire it up in app.js.
#   [write index.html]
#   [write app.js]
# Muse: Done. Want me to persist the choice in localStorage?
# > You: yes
```

The chat model also has a **`remember`** tool and a **memory file** — it stores durable
facts you tell it (preferences, project conventions, toolchain choices) in
`.aicli/memory.md` (per-project) or `~/.aicli/memory.md` (global), and reads them back
at the start of every session:

```bash
# > You: remember that this project deploys with `make flash`
# Muse: Noted — I'll use `make flash` when you ask me to deploy.
```

Commands inside chat: `/quit` (or `/exit`), `/clear` (reset conversation history),
`/help`. Pass an opening message directly: `aagent --chat "what's in this repo?"`.
Chat uses the builder model and the configured autonomy `--mode`; each turn is capped
at `CHAT_MAX_STEPS` tool calls (default 15) and the prompt keeps the last
`CHAT_HISTORY_TURNS` turns (default 10). Every session starts with a snapshot of the
working directory (file listing, 2 levels deep), so the model knows what already
exists and inspects files itself instead of asking you to describe them. The system
prompt also teaches the turn protocol with a worked example (narrate → tool calls →
summary), plus rules the one-shot agent doesn't need: always `read` before
overwriting, announce the file list before multi-file work, and never paste file
contents into chat — always use the `write` tool.

### `aicli model`

Manage the Ollama model `aicli` talks to. The choice is saved in `config.env`, so it
persists across sessions.

```bash
aicli model              # show active model + downloaded models
aicli model llama3:8b    # switch; offers to `ollama pull` it if not downloaded
aicli model -m qwen2.5-coder:32b
```

### `aicli config`

View or change settings without editing files by hand.

```bash
aicli config                         # show all settings
aicli config CHAT_MODEL llama3:8b    # same effect as `aicli model llama3:8b`
aicli config context-window 10       # keep the last 10 files as build context
```

---

## Configuration

Settings live in `config.env` next to the script. `aicli model` and `aicli config` edit it
for you (one key at a time — other keys are preserved).

| Key | Default | Meaning |
|---|---|---|
| `CHAT_MODEL` | `qwen2.5-coder:7b` | Ollama model used by every subcommand. |
| `CONTEXT_WINDOW_FILES` | `5` | In `build` mode, how many of the most recently generated files are fed back as context for the next file. |
| `ARCHITECT_MODEL` | _(follows `CHAT_MODEL`)_ | Model that plans in `agent` mode. |
| `BUILDER_MODEL` | _(follows `CHAT_MODEL`)_ | Model that runs the tool loop in `agent` mode. |
| `REVIEWER_MODEL` | _(follows `CHAT_MODEL`)_ | Model that critiques in `agent` mode. |
| `AGENT_MODE` | `sandbox` | Agent autonomy: `sandbox` (bwrap jail), `confirm` (ask per command), `dry-run` (execute nothing). |
| `AGENT_MAX_STEPS` | `25` | Max tool-call turns per agent run, across all phases. |
| `AGENT_MAX_REVISIONS` | `2` | Reviewer → builder fix rounds per run. |
| `AGENT_MAX_OUTPUT_CHARS` | `4000` | Truncate individual tool outputs to this many chars. |
| `AGENT_CMD_TIMEOUT` | `60` | Seconds before a single agent shell command is killed. |
| `AGENT_MODEL_TIMEOUT` | `300` | Seconds before a single agent model call is killed. |
| `AGENT_HISTORY_KEPT` | `6` | How many of the most recent builder tool exchanges are kept verbatim in each prompt (the deliverables checklist carries the rest of the state). |
| `CHAT_HISTORY_TURNS` | `10` | How many conversation turns are kept in each `--chat` prompt. |
| `CHAT_MAX_STEPS` | `15` | Max tool calls per `--chat` turn before the model must reply. |

**Tuning `CONTEXT_WINDOW_FILES`:** higher values give the model more cross-file awareness
(better imports, consistent config keys) at the cost of longer prompts and slower generation.
On large projects (15+ files), a high value can exceed the model's context window — if
builds start producing truncated or confused files, lower it.

---

## How `build` works under the hood

```
 your one-line project description
              │
              ▼
 ┌─────────────────────────┐
 │ PHASE 1 — Architect     │  model is told: "decompose this into files,
 │                         │  OUTPUT ONLY a raw JSON array [{path, description}]"
 └────────────┬────────────┘
              │  <think> blocks / fences stripped, JSON validated with python3
              │  (up to 3 attempts with increasingly strict instructions)
              ▼
 ┌─────────────────────────┐
 │ PHASE 2 — Build loop    │  for each file in the plan:
 │                         │   1. path is sanitized (relative-only, no "..")
 │                         │   2. prompt = role + file requirements + project
 │                         │      request + last N generated files as context
 │                         │   3. model writes code inside <FILE_CONTENT> tags
 │                         │   4. tags/fences stripped, file written to disk,
 │                         │      chmod +x if it's a script, appended to context
 └────────────┬────────────┘
              ▼
     project directory on disk
```

Key implementation details:

- **Prompts are piped via stdin** (`ollama run "$model" <<< "$prompt"`), never through
  unquoted heredocs, so text coming back from the model can't be re-expanded as shell code.
- **The JSON plan is parsed once** into Bash arrays (NUL-delimited, so multi-line
  descriptions can't desynchronize paths from their descriptions).
- **Code extraction** looks for `<FILE_CONTENT>…</FILE_CONTENT>` and falls back to the
  whole response if the model forgets the tags.
- **Generated code is kept verbatim** — blank lines and formatting are preserved as the
  model wrote them.

---

## Security notes

- `build` writes files to disk based on model output. Paths are strictly validated:
  absolute paths and any `.`/`..`/empty path segment abort the build. The tool only
  ever writes inside the directory you run it from.
- `agent` goes further because it can also *run* commands. Defenses, in order:
  1. every `read`/`write`/`ls` path is jailed to the project directory;
  2. a denylist rejects `sudo`, `su`, `mkfs`, `dd`, `rm -rf /`, fork bombs, etc. —
     these never execute, in any mode;
  3. in `sandbox` mode commands run under `bwrap` with the project dir read-write,
     the system read-only, and no network or home directory visible;
  4. in `confirm` mode every command needs your explicit `y`;
  5. every command is killed after `AGENT_CMD_TIMEOUT` seconds.
- Model output is never `eval`'d or executed by `aicli` itself — but the files it
  *writes* are code, so review them before running, same as any generated code.
- Everything runs locally through Ollama. Your prompts never leave the machine
  (unless you explicitly pick a remote Ollama host).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Error: required command 'ollama' was not found` | Ollama not installed / not on PATH | Install from [ollama.com](https://ollama.com) and open a new shell |
| `Error: required command 'python3'/'perl' was not found` | Missing interpreter | Install via your package manager (`apt install python3 perl`, `brew install python`, …) |
| `ai "..."` prints the help text | The `ai` symlink wasn't created (Method A) or `~/.local/bin` isn't on PATH | Re-run the `ln -sf` steps; check `echo $PATH` |
| `could not connect to localhost:11434` | Ollama daemon isn't running | Run `ollama serve` (or start the Ollama app) |
| `Architect failed to generate a valid JSON plan after 3 attempts` | Small/weaker model struggling with the JSON-only instruction | Switch to a stronger model (`aicli model qwen2.5-coder:32b`) or simplify the request |
| Builds get confused on large projects | Context window exhausted | Lower the context window: `aicli config context-window 3` |
| `Error: architect returned an unsafe file path` | Model emitted an absolute/`..` path | Re-run the build; if it persists, try a different model |
| `AGENT: max steps reached` (without `done`) | Task too big for `AGENT_MAX_STEPS`, or the model is looping | Raise it for the run (`--max-steps 50`) or split the task |
| `done REJECTED - these deliverables are still missing` (agent) | The builder tried to finish before creating every file on the checklist | Nothing to fix — the gate worked. If it repeats, the task may be too vague; make the deliverables explicit in your request |
| `AGENT: model ignored the tool protocol 3 times in a row` | Small model replying in prose instead of `<TOOL>` blocks | Use a larger model for the builder role (`aicli config builder-model qwen2.5-coder:14b`) |
| `AGENT: revision round produced no changes` | The builder couldn't act on the reviewer's notes | Check the notes in the output; simplify the task or use a stronger builder model |
| `Warning: 'bwrap' not found; ... Falling back to confirm mode` | `bwrap` isn't installed | Install bubblewrap (`apt install bubblewrap`) for unattended sandbox mode, or just use `confirm` |
| `Error: command denied by safety policy` (agent) | The model tried a denylisted command (`sudo`, `rm -rf /`, …) | Nothing to fix — the guard worked. The agent sees the denial and adapts |
| `Muse: (stopping - too many tool steps this turn; ...)` (chat) | One chat turn needed more than `CHAT_MAX_STEPS` tool calls | Ask for the work in smaller pieces, or raise it: `aicli config chat-max-steps 25` |
| `Error: model 'X' is not downloaded` (agent) | A role model isn't pulled | `ollama pull X`, or point the role at a model you have |

---

## Project layout

```
aicli            Main script - all subcommands live here
ai               Wrapper: shortcut for `aicli ask`
aarch            Wrapper: shortcut for `aicli arch`
abuild           Wrapper: shortcut for `aicli build`
aagent           Wrapper: shortcut for `aicli agent`
config.env       Settings (model name, context window) - edited via `aicli model|config`
CONTRIBUTING.md  How to contribute
LICENSE          MIT
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Bug reports and pull requests are welcome —
please run `shellcheck` and `bash -n` on the script before submitting.

---

## Versioning

This project follows [Semantic Versioning](https://semver.org). Check the repository
tags for releases (current: v2.9.6).

---

## License

MIT — see [LICENSE](LICENSE). Free for local development and any other use the
license permits.
