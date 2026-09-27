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
| `aicli agent` / `aagent` | Sandboxed multi-model agent: an architect model plans, a builder model works the filesystem with tools (`read`/`write`/`ls`/`run`/`done`), and a reviewer model critiques the result and sends it back for fixes. |
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

1. **Architect** (`ARCHITECT_MODEL`) writes a short step-by-step plan.
2. **Builder** (`BUILDER_MODEL`) works the plan as a tool-calling loop: it emits one tool
   call per turn — `read`, `write`, `ls`, `run`, `done` — and `aicli` executes it and feeds
   the result back, up to `AGENT_MAX_STEPS` turns.
3. **Reviewer** (`REVIEWER_MODEL`) reads every file the builder changed, then either approves
   or sends concrete revision notes back to the builder (up to `AGENT_MAX_REVISIONS` rounds).

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

Tip: give the roles different models for the classic setup — a larger reasoning model to
architect and review, a fast coder model to do the writing:

```bash
aicli config architect-model llama3.1:8b
aicli config builder-model qwen2.5-coder:7b
aicli config reviewer-model llama3.1:8b
```

(Each role falls back to `CHAT_MODEL` when unset, so this is opt-in.)

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
| `Warning: 'bwrap' not found; ... Falling back to confirm mode` | `bwrap` isn't installed | Install bubblewrap (`apt install bubblewrap`) for unattended sandbox mode, or just use `confirm` |
| `Error: command denied by safety policy` (agent) | The model tried a denylisted command (`sudo`, `rm -rf /`, …) | Nothing to fix — the guard worked. The agent sees the denial and adapts |
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
tags for releases (current stable: v2.3.1).

---

## License

MIT — see [LICENSE](LICENSE). Free for local development and any other use the
license permits.
