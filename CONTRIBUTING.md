# Contributing to aicli

Thanks for your interest in contributing! Bug reports, feature ideas, and pull
requests are all welcome.

## Getting set up

1. Fork the repo and clone your fork:
   ```bash
   git clone https://github.com/<you>/ai-cli-toolkit.git
   cd ai-cli-toolkit
   ```
2. Make sure you have the [requirements from the README](README.md#requirements):
   Bash 4+, Ollama with at least one model pulled, `python3`, and `perl`.
3. Install using either method in the README, pointing at your clone.

No build step, no dependencies to install — it's a Bash script. Edit and run.

## Code style

- The whole tool is one Bash script (`aicli`) plus three tiny wrappers. Keep it that way
  unless a change genuinely needs a new file.
- Target **Bash 4+** (no Bash 5-only features without a fallback).
- `set -uo pipefail` is on: quote your variables, and use `${var:-}` / `${1:-}` for
  anything that may be unset.
- Functions that read stdin are fine, but keep the `strip_model_noise` /
  `sanitize_relpath` helpers as the single place their logic lives — don't
  re-implement them inline.
- Keep the help text (`show_help`) and the README's command reference in sync
  whenever you add or change a subcommand or flag.

## Before you submit

1. **Syntax check:** `bash -n aicli ai aarch abuild`
2. **Lint:** `shellcheck -S warning aicli ai aarch abuild` — fix warnings, or
   explain in the PR why one is a false positive.
3. **Smoke test** with Ollama running:
   ```bash
   aicli --help && aicli --version
   aicli model            # should list your models
   ai "Reply with the word OK."
   mkdir /tmp/aicli-smoke && cd /tmp/aicli-smoke
   abuild "A 2-file Bash tool: config/app.conf and bin/app"
   ls -R                  # both files exist, bin/app is executable
   ```
4. **Update docs:** README's command reference, configuration table, and
   troubleshooting section should reflect your change. Docs-only PRs are welcome too.

## Pull request checklist

- [ ] `bash -n` and `shellcheck` are clean
- [ ] Smoke test passes (or note which parts you couldn't run and why)
- [ ] README updated if behavior, flags, or config changed
- [ ] One focused change per PR — separate PRs for separate ideas

## Reporting bugs

Open an issue with:

- `aicli --version` output
- Your OS and Bash version (`bash --version`)
- The exact command you ran and the full output (redact anything private)
- `ollama list` output if the issue involves model behavior

## Feature ideas

The roadmap is informal — if an idea needs discussion first, open an issue before
writing code. Rough areas of interest: streaming output, per-project config
overrides, a `--dry-run` mode for `build`, and retry/backoff for transient
Ollama failures.
