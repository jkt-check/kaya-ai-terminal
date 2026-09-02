# kaya FAQ

> Full usage instructions: [User Guide](docs/PRODUCT.md) (bilingual). Privacy design details: [docs/PRIVACY.en.md](docs/PRIVACY.en.md). Per-version changes: [Release Notes](RELEASE_NOTES.en.md).
>
> [中文版](FAQ.md)

## Contents

- [Installation & Platforms](#installation--platforms)
- [Privacy & Security](#privacy--security)
- [Models & Configuration](#models--configuration)
- [Terminal & Tabs](#terminal--tabs)
- [Coding-agent Collaboration (coding mode)](#coding-agent-collaboration-coding-mode)
- [IM Channels (Feishu / WeCom)](#im-channels-feishu--wecom)
- [Troubleshooting](#troubleshooting)

---

## Installation & Platforms

**Which platforms are supported?**

macOS (universal dmg for Apple Silicon and Intel) and Windows x64 (msi / nsis installers; nsis recommended). Linux is not supported yet. Installers are downloaded from the [official kaya download page](https://yunhou.ai/download).

**macOS says it "cannot verify the developer" — what do I do?**

Right-click `kaya.app` in Finder → "Open", or allow it under System Settings → Privacy & Security. Do not disable Gatekeeper.

**Why does macOS ask "Allow kaya to access Desktop / Downloads / Documents"?**

This is **normal macOS behavior**, not kaya sneakily requesting permissions. When the AI runs a shell command (e.g. `ls ~/Desktop`), kaya spawns a separate subprocess, and macOS TCC prompts the first time an app touches a protected folder — the same happens with iTerm2, WezTerm, VS Code Terminal, and every app that spawns a system shell.

- Click "Allow" once and that folder won't prompt again (each new folder still asks once — that's macOS by design and cannot be removed at the app layer).
- If you let the AI run shell commands often, grant kaya Full Disk Access once (System Settings → Privacy & Security → Full Disk Access → add kaya.app). kaya's 5 privacy gates still ensure the LLM cannot see your file contents by default; FDA only lets kaya's terminal subprocesses work normally.
- Since macOS Sequoia, the system re-asks for Screen Recording permission roughly monthly — that's an OS behavior every desktop app sees.

**Which shell is used on Windows?**

PowerShell 7 (pwsh) by default, falling back to the built-in Windows PowerShell 5.1; git-bash users can keep their POSIX environment via `$SHELL`. Shortcuts map to the Ctrl+Shift family on Windows (the ⌘ family on macOS).

---

## Privacy & Security

**Can the AI read my files?**

By default, **no**. Content-reading commands (`cat`, `head`, `grep`, `less`, …) are rejected at the policy layer — this is structural, not a setting. The only exception: files you deliberately place under `~/.kaya/` (the trusted root) can be read with read-only commands — scrubbing still applies and all write operations remain blocked. See the user guide chapter "Let the AI read a file" in [docs/PRODUCT.md](docs/PRODUCT.md).

**Will a key I paste be sent out?**

No. Your input is scrubbed before it enters the conversation and logs (full PEM-block redaction, API keys, ID numbers, phone numbers, credit cards — 30+ pattern families), and the UI shows a "scrubbed" badge on hits. But scrubbing is not an absolute guarantee — never paste truly important keys into any LLM app.

**What does the cloud model receive?**

Only your (scrubbed) chat text and (scrubbed) tool results, sent to the model source you selected in Settings:

- **Built-in model (default)**: sent over your login token to the yunhou first-party server, which answers on your behalf;
- **OpenAI compatible (your own key)**: sent to the LLM provider you chose (DeepSeek / OpenAI / OpenRouter / etc.);
- **Ollama (local)**: nothing leaves your machine.

Your local files, API key contents, and environment variables are never sent. Local UI state, sessions, and audit logs live only on your machine (`localStorage` + `~/.kaya/`) — no cloud sync, no telemetry.

**Where is my API key stored?**

Only on your machine. It is sent to your chosen LLM provider in the authentication header of each request, for that provider's authentication — never to any other party. kaya does not promise "never uploaded": any network request needs authentication, and authentication needs the key.

**Can the AI delete my files?**

The AI has no permission to modify or delete your files. Destructive commands (`rm -rf`, `sudo`, `dd`, `chmod 777`, …) are blocked at the safety gate; `curl | bash`, `brew install`, environment dumps, and the like are never executed. AI-invoked commands run in isolated subprocesses, never in your terminal; blocked commands show the specific reason in the UI.

**How do I audit what the AI did?**

Every session automatically writes a JSONL audit log (`~/.kaya/audit/`). Settings → Audit lets you replay each session's full event stream and run a three-layer evaluation (rule replay + anomaly rules + AI judge).

---

## Models & Configuration

**Do I need my own API key?**

No. The default is the **built-in model**: sign in with WeChat and you're ready — no key needed. You can also switch to "OpenAI compatible" with your own key (DeepSeek / OpenAI / OpenRouter / Ollama / custom endpoint), or pick Ollama for fully local inference.

**Which providers are supported?**

Five built-in presets: DeepSeek, OpenAI, OpenRouter, Ollama, and Custom (OpenAI compatible). For Moonshot / SiliconFlow / Zhipu GLM / self-hosted vLLM, choose "Custom" and fill in their OpenAI-compatible endpoint. Since v2.1.8 you can save multiple model profiles and switch from the top bar.

**What is Thinking Mode?**

When enabled, the AI shows a collapsible reasoning block before its final answer; the reasoning is also recorded in the audit log. The cost: slower responses and more tokens. Note: the "Reasoning Model" field in Settings currently only feeds the audit AI judge — to change the model used for chat, edit the "Model" field.

**Why was a command rejected?**

The command the AI attempted may be: outside the allowlist (denied by default), destructive, interactive (vim / REPLs), or system-modifying (`sudo` / `brew install`). The UI shows the specific reason. Commands you type into the terminal yourself are never gated — that area is your own responsibility.

---

## Terminal & Tabs

**Is each tab an independent shell?**

Yes. Every tab gets its own PTY, loads your `.zshrc` (or your PowerShell profile on Windows), and matches your native terminal's environment, aliases, and conda setup. Each tab has its own working directory, command history, and scrollback; on close, its history merges back into your global shell history.

**Common shortcuts?**

macOS: ⌘T new tab, ⌘W close, ⌘1–9 switch tabs, ⌘D / ⌘⇧D split, ⌘⌥arrow to move between panes, ⌘⇧F focus mode, ⌘0/=/- font size. On Windows these map to the Ctrl+Shift family. Full list in the user guide's Shortcuts chapter ([docs/PRODUCT.md](docs/PRODUCT.md)).

**A tab disconnected — how do I recover?**

Settings → General → Shell → "Reconnect now".

---

## Coding-agent Collaboration (coding mode)

**What is coding mode?**

Turn on the coding-mode toggle in the top bar and kaya actively watches the coding agents (Claude Code / Kimi Code / OpenCode / Reasonix, …) running in your tabs: it relays confirmation prompts to you and summarizes progress when a task finishes. Monitoring is read-only — decisions are always yours.

**Can the AI operate my terminal directly?**

Not by default. The AI is an **observer** of your terminal (reads output only), not an operator. The single exception: with the coding-mode master switch on, a supported AI CLI in the tab's foreground, and your per-item approval on the grant card, the AI may type into that tab on your behalf. Grants bind to the current process and expire when it exits or the tab closes; you can revoke anytime. Your regular shell prompt never accepts AI input.

**How do `/agents` and `/` commands work?**

Type `/` in the chat input to open the command menu: `/agents` lists and binds running agents, `/new` starts a new conversation, `/sessions` browses history, `/help` shows help. Once bound, non-built-in `/commands` (e.g. `/auto`) are forwarded straight to that agent's input line without consuming an LLM turn.

**What does the Agents management center do?**

Silently installs / updates / uninstalls coding agents in the background (npm > brew > official script), and keeps multiple model-provider profiles per agent, written into each agent's own config file with one click (automatic .bak backup first). You can also just say "install Claude Code for me" in the chat.

---

## IM Channels (Feishu / WeCom)

**Can I use kaya from Feishu?**

Yes (since v2.1.0). Settings → IM Channels — the recommended path is "Connect Feishu by scanning a QR code", which completes the setup in one step. Once running, message the bot in Feishu and your local kaya processes it and streams the reply — direct long-connection, no public IP needed. The WeCom entry is currently greyed out ("coming soon").

**Can strangers message the bot?**

No. The admission allowlist rejects everything from outside the list by default, and the channel won't start with an empty list. IM messages go through the same five-gate privacy pipeline as desktop chat, and platform credentials live only in the system keychain.

---

## Troubleshooting

**The AI never responds / answers very slowly?**

- Streaming responses time out automatically after 30 s of silence; press Esc or ⏹ to interrupt manually.
- Thinking Mode adds latency; check your network or switch to a faster model in Settings.
- The first LLM connection allows up to a 30 s handshake; on "LLM connection timeout", check your network and API key, then retry (since v2.1.11 weak networks retry automatically).

**The terminal is unresponsive?**

Press Esc to interrupt the current turn → Settings → General → Shell → "Reconnect now" → if it's still stuck, close the tab with ⌘W and open a new one with ⌘T.

**Lost sessions?**

Check whether `~/.kaya/conversations/` was cleaned up. To wipe all local data: `rm -rf ~/.kaya/`.

**Where are the diagnostic logs?**

Settings → App Log → "Open log folder"; logs live under `~/.kaya/logs/` (they do not contain your conversation content).

**Anything else?**

Open an [Issue](../../issues) in this repository.
