# `aicli` — Unified Local AI Developer Suite

`aicli` is a lightweight, offline AI command-line suite powered by Ollama. It unifies single-prompt questions, high-level system design blueprinting, autonomous multi-file code generation, and dynamic local model switching into a single, seamless terminal tool.

---

## Features

* **100% Offline & Private:** Runs entirely on your local machine using Ollama—no cloud APIs or data tracking.
* **Autonomous Multi-File Generation:** Automatically plans and builds complex multi-file project directory structures from a single prompt.
* **Context-Aware Execution:** Built-in cumulative context memory ensuring clean integration across generated files (such as relative path resolution).
* **Multi-Alias Support:** Fast shortcuts for different workflows (`ai`, `aarch`, `abuild`).
* **Dynamic Model Switching:** Easily swap between your preferred local models on the fly.

---

## Installation & Quick Start

Set up your symlinks to make the suite globally accessible in your terminal:

```bash
mkdir -p ~/.local/bin
ln -sf ~/aicli/aicli ~/.local/bin/aicli
ln -sf ~/aicli/aicli ~/.local/bin/ai
ln -sf ~/aicli/aicli ~/.local/bin/aarch
ln -sf ~/aicli/aicli ~/.local/bin/abuild

(Ensure that ~/.local/bin is added to your system's PATH variable in your .bashrc or .zshrc if it isn't already).
Subcommands & Usage
1. aicli model

Switch or view your active Ollama model.
Bash

aicli model qwen2.5-coder:7b

2. aicli ask (ai)

Your quick shell assistant and offline command-line Q&A tool.
Bash

ai "How do I find all files modified in the last 24 hours?"

3. aicli arch (aarch)

Generate architectural blueprints and file trees for planned applications before building.
Bash

aarch "Design a modular Python web scraper with a config parser and database logger."

4. aicli build (abuild)

An autonomous multi-file code generator that maps out a project architecture and writes out all files recursively.
Bash

abuild "A simple counter CLI tool with config settings, a core logic library, and a main executable script."

Versioning

This project follows Semantic Versioning. Check the repository tags for specific releases (current stable: v2.3.0).
License

This project is open-source and available for local development use.
