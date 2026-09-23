# kaya — Mission Control for AI Coding Agents

> **Mission Control for AI Coding Agents · Privacy First · Fast · Simple & Effective**

> [中文 README](README.md)

**kaya** is a Mission Control for AI coding agents that runs on **your own Mac / Windows PC**: chat and code in one window — the LLM can run shell commands and read terminal output, and with coding-mode grants it can type into coding agents (Claude Code / Kimi Code / OpenCode / Codex) running in tabs. Type `@` or `/agents` in chat to bind a conversation to a running coding agent; each tab gets its own PTY, so multiple agents run truly in parallel. Away from your computer? Use kaya through the Feishu (since v2.1.0) / Slack (since v2.3.6) integration (WeCom coming soon) or browser remote access (since v2.3.2, cloud relay, explicitly opt-in).

**Privacy first**: by default the LLM cannot see any of your file contents — this is structural, not a setting. Every user input and tool output passes through 5 gates (input guard → policy gate → safety gate → isolated execution → output scrub) before leaving your machine. All state stays local; no cloud sync. When signed in, kaya collects usage statistics that never include conversation content (active days, usage duration, platform and version distribution), and you can turn this off anytime in Settings → Account; nothing is reported while signed out.

## What This Repository Is

This is kaya's **public documentation repository** — published content only, **no source code**:

| Content | 中文 | English |
|---|---|---|
| User guide | [docs/PRODUCT.md](docs/PRODUCT.md) | [docs/PRODUCT.en.md](docs/PRODUCT.en.md) |
| Privacy design | [docs/PRIVACY.md](docs/PRIVACY.md) | [docs/PRIVACY.en.md](docs/PRIVACY.en.md) |
| FAQ | [FAQ.md](FAQ.md) | [FAQ.en.md](FAQ.en.md) |
| Release notes | [RELEASE_NOTES.md](RELEASE_NOTES.md) | [RELEASE_NOTES.en.md](RELEASE_NOTES.en.md) |

## Download & Install

Installers are published on the **[official kaya download page](https://www.yunhouai.com/download)**:

- **macOS**: download `kaya-<version>-macos-universal.dmg` (universal for Apple Silicon and Intel), then drag `kaya.app` into `/Applications`.
- **Windows (x64)**: download `kaya-<version>-windows-x64.exe` (nsis, recommended) or `.msi`, and run the installer.

See the [user guide — Install kaya](docs/PRODUCT.en.md#install-kaya) and the per-version [release notes](RELEASE_NOTES.en.md) for details.

## Feedback & Issues

- Check the [FAQ](FAQ.en.md) and the "FAQ / Troubleshooting" chapters of the [user guide](docs/PRODUCT.en.md) first.
- Still stuck? Open an [Issue](../../issues) in this repository.

## License

Documentation in this repository is licensed under [CC BY 4.0](LICENSE). The kaya software itself is licensed as shown in the in-app About dialog.
