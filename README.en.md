# kaya · The AI terminal built for AI developers

> **The AI terminal built for AI developers**

> [中文 README](README.md)

**kaya** is **the AI terminal built for AI developers**, running on **your own Mac / Windows PC**. It brings model management, coding-agent management, a security gateway and an AI brain into one full terminal workspace.

## Core Capabilities and Full Terminal

| Core capability | Current scope |
|---|---|
| **Models and providers** | A unified provider-to-model library, with a 200+ provider catalog that pre-fills endpoints and models; your own API keys, provider OAuth, custom compatible endpoints and local Ollama models, subject to the credential and protocol coverage in the user guide |
| **Coding-agent management** | Install, configure and update Claude Code / Codex / OpenCode / Kimi Code, bind models, and inspect activity and terminal output in independent tabs |
| **Security gateway** | An optional local model gateway, off by default; only explicitly connected coding-agent model requests are checked, blocked or scrubbed according to policy, with supported replacements restored when possible and protection records available |
| **AI brain** | Use terminal context to explain commands, errors and agent plans, read progress, and summarize results and next steps; with coding mode and per-tab approval, send instructions to supported agents and hand off tasks and context |

**A full terminal is the foundation**: zsh on macOS and PowerShell on Windows, your shell profiles, aliases, scripts and command-line tools, independent PTYs, multiple tabs and panes. Your own terminal commands and the AI's restricted tool execution are separate paths.

**Core does not mean exhaustive**: type `@` or `/agents` to bind running agents and work with multiple agents in parallel. The cross-agent ToDo List, cross-device coordination, Feishu / Slack (WeCom coming soon), browser remote access, memory, scheduled tasks, web search, image input, session history, usage statistics, privacy audit, config optimization and auto-update remain part of the product. The core grouping changes no support status, subscription requirement or authorization boundary. Privacy First, speed and simplicity remain product-wide principles. See the [user guide](docs/PRODUCT.en.md).

## Privacy and Security

**Privacy first**: the LLM cannot freely read local file contents. Restricted tools may read files you deliberately place under the `~/.kaya/` trusted root, except sensitive control-plane data excluded by policy. Chat is protected by the input guard, policy gate, safety gate, isolated execution and output scrub. AI tools run in separate subprocesses, never by typing into your regular shell prompt.

The model gateway is separate from chat, is not a system-wide transparent proxy, and does not protect direct traffic from agents that have not been connected. API keys and OAuth tokens live in the system keychain and are used only for the relevant service's authentication; gateway-managed configs contain only loopback URLs, placeholder tokens and model aliases. Gateway privacy records can contain matched originals and replacement tokens; they stay local, are masked by default, and are not automatically included in model context or diagnostic exports.

Sessions, memory, audit and settings stay local without automatic cloud sync. Cloud models receive the relevant content, and explicitly enabled search, IM, remote access and cross-device coordination have their own transmission paths. Signed-in usage heartbeats can be disabled; **cloud token-usage upload is off by default and requires explicit consent**, and never uploads conversation or tool content. Local Ollama inference sends no inference or conversation / tool content out, but does not automatically disable these independent channels. See the [privacy design](docs/PRIVACY.en.md) for the full boundaries.

## What This Repository Is

This is kaya's **public documentation repository** — published content only, **no source code**:

| Content | 中文 | English |
|---|---|---|
| User guide | [docs/PRODUCT.md](docs/PRODUCT.md) | [docs/PRODUCT.en.md](docs/PRODUCT.en.md) |
| Privacy design | [docs/PRIVACY.md](docs/PRIVACY.md) | [docs/PRIVACY.en.md](docs/PRIVACY.en.md) |
| FAQ | [FAQ.md](FAQ.md) | [FAQ.en.md](FAQ.en.md) |
| Release notes | [RELEASE_NOTES.md](RELEASE_NOTES.md) | [RELEASE_NOTES.en.md](RELEASE_NOTES.en.md) |

## Download & Install

Installers are published on the **[official kaya download page](https://yunhou.ai/download)**:

- **macOS**: download `kaya-intl-v<version>-macos-universal.dmg` (universal for Apple Silicon and Intel), then drag `kaya.app` into `/Applications`.
- **Windows (x64)**: download `kaya-intl-v<version>-windows-x64.exe` (nsis, recommended) or `.msi`, and run the installer.

See the [user guide — Install kaya](docs/PRODUCT.en.md#install-kaya) and the per-version [release notes](RELEASE_NOTES.en.md) for details.

## Feedback & Issues

- Check the [FAQ](FAQ.en.md) and the "FAQ / Troubleshooting" chapters of the [user guide](docs/PRODUCT.en.md) first.
- Still stuck? Open an [Issue](../../issues) in this repository.

## License

Documentation in this repository is licensed under [CC BY 4.0](LICENSE). The kaya software itself is licensed as shown in the in-app About dialog.
