# kaya FAQ

> Full usage instructions: [User Guide](docs/PRODUCT.en.md). Privacy design details: [docs/PRIVACY.en.md](docs/PRIVACY.en.md). Per-version changes: [Release Notes](RELEASE_NOTES.en.md).
>
> [中文版](FAQ.md)

## Contents

- [Installation & Platforms](#installation--platforms)
- [Privacy & Security](#privacy--security)
- [Models & Configuration](#models--configuration)
- [Terminal & Tabs](#terminal--tabs)
- [Coding-agent Collaboration (coding mode)](#coding-agent-collaboration-coding-mode)
- [IM Channels (Feishu / Slack / WeCom)](#im-channels-feishu--slack--wecom)
- [Remote Access (browser)](#remote-access-browser)
- [Cross-Device Coordination](#cross-device-coordination)
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

By default, **not arbitrarily**. Files you deliberately place under the `~/.kaya/` trusted root can be read by restricted tools, with output scrubbing and additional sensitive-read rules for control-plane data. Deleting, moving and directly editing files remain blocked; restricted output redirection to safe file extensions is an exception. AI tools run in isolated subprocesses, separately from commands you run in your real terminal. See [Let the AI Read a File](docs/PRODUCT.en.md#let-the-ai-read-a-file).

**Will a key I paste be sent out?**

Recognized matches are scrubbed before entering the conversation and logs (full PEM-block redaction, API keys, national IDs, phone numbers, credit cards and other patterns), and the UI shows a "scrubbed" badge. Detection is not an absolute guarantee that every key stays local; never paste truly important keys into any LLM app.

**What does the cloud model receive?**

Chat text and tool results are scrubbed and sent to your selected model. Images you deliberately paste are re-encoded and stripped of metadata, then sent separately without text scrubbing; the built-in model does not support images:

- **Built-in model (default)**: sent over your login token to the yunhou first-party server, which answers on your behalf;
- **OpenAI compatible (your own key)**: sent to the LLM provider you chose (DeepSeek / OpenAI / OpenRouter / etc.);
- **Ollama (local)**: local inference and conversation / tool content stay on the machine, but independent analytics, search, IM and remote channels are not automatically disabled.

Files are not uploaded arbitrarily; file content you provide or scrubbed terminal output can enter model context, and API keys are used only for the relevant service's authentication. Local UI state, sessions, memory and audit are not automatically cloud-synced. Signed-in usage heartbeats can be disabled in Settings → Account. **Cloud token-usage upload is a separate feature, off by default and requiring explicit consent**; it uploads aggregate counts and device-registration fields, not conversation or tool content. Explicitly enabled search, IM, remote access and cross-device coordination also have their own transmission paths. See the [privacy design](docs/PRIVACY.en.md).

**Where is my API key stored?**

Only in your machine's **system keychain** (macOS Keychain / Windows Credential Manager, since v2.2.1; local configuration from earlier versions is migrated automatically). It is sent to your chosen LLM provider in the authentication header of each request, for that provider's authentication — never to any other party, and never written to the audit log. kaya does not promise "never uploaded": any network request needs authentication, and authentication needs the key.

**Can the AI delete my files?**

kaya's built-in AI `run_shell` cannot delete, move or directly edit your files; restricted redirection and authorized config management have their own explicit boundaries. A coding agent you run in the terminal has its own file permissions and confirmations: kaya's tool allowlist is not a global sandbox around that agent. Blocked operations show the reason in the UI.

**How do I audit what the AI did?**

Desktop chat and scheduled turns wired into auditing write JSONL under `~/.kaya/audit/`, including scrubbed text; Settings → Audit provides replay and three-layer evaluation. IM and remote message auditing is not fully wired up. Gateway request events are content-free metadata; gateway privacy records can contain matched originals and tokens and are a separate sensitive local store.

---

## Models & Configuration

**Do I need my own API key?**

Not necessarily. Built-in models need no personal API key, but require sign-in and an active subscription: WeChat sign-in for the China build, GitHub for the international build. You can also add a BYOK provider or local Ollama under Settings → Models. Sign-in allows free BYOK chat; an active paid subscription unlocks built-in models, coding mode, IM channels and remote access.

**Which providers are supported?**

The unified library has two levels, provider → model, with a compiled-in catalog of 200+ providers, custom compatible endpoints, and multiple accounts per provider. Endpoints and models can be pre-filled; online discovery follows user actions, without background polling. Credentials support API keys and enabled provider OAuth flows. ChatGPT subscriptions serve only the four coding agents through the gateway, not kaya's own Chat or audit AI judge. See [Model Library and Providers](docs/PRODUCT.en.md#model-library-and-providers).

**What is the security gateway? Does it take over all traffic by default?**

The security gateway is the local model gateway, off by default and covering only explicitly connected coding-agent model requests; chat remains independent. Agent configs contain only loopback URLs, placeholder tokens and aliases, with real credentials in the keychain and gateway memory. Default strict mode does not automatically rewrite ordinary PII; policy-based detection and response restoration are not absolute guarantees. See [Model Gateway](docs/PRODUCT.en.md#model-gateway).

**What is Thinking Mode?**

When supported and enabled, the AI shows collapsible reasoning segments at the cost of more time and tokens. Chat reasoning parameters follow the chat-endpoint switch and current model; per-model thinking declarations and efforts primarily configure agent projections. Legacy `reasoning_model` data is used only by the audit AI judge, and no longer has an editor in Settings.

**Why was a command rejected?**

The command the AI attempted may be: outside the allowlist (denied by default), destructive, interactive (vim / REPLs), or system-modifying (`sudo` / `brew install`). The UI shows the specific reason. Commands you type into the terminal yourself are never gated — that area is your own responsibility.

---

## Terminal & Tabs

**Is each tab an independent shell?**

Yes. Every tab gets its own PTY, loads your `.zshrc` (or your PowerShell profile on Windows), and matches your native terminal's environment, aliases, and conda setup. Each tab has its own working directory, command history, and scrollback; on close, its history merges back into your global shell history.

**Common shortcuts?**

macOS: ⌘T new tab, ⌘W close, ⌘1–9 switch tabs, ⌘D / ⌘⇧D split, ⌘⌥arrow to move between panes, ⌘⇧F focus mode, ⌘0/=/- font size. On Windows these map to the Ctrl+Shift family, with exceptions detailed in [Keyboard Shortcuts](docs/PRODUCT.en.md#keyboard-shortcuts).

**A tab disconnected — how do I recover?**

Settings → General → Shell → "Reconnect now".

---

## Coding-agent Collaboration (coding mode)

**What is coding mode?**

Turn on the coding-mode toggle in the top bar and kaya actively watches the coding agents (Claude Code / Kimi Code / OpenCode / Codex, …) running in your tabs: it relays confirmation prompts to you and summarizes progress when a task finishes. Monitoring is read-only — decisions are always yours.

**Can the AI operate my terminal directly?**

Not by default. The AI is an **observer** of your terminal (reads output only), not an operator. The single exception: with the coding-mode master switch on, a supported AI CLI in the tab's foreground, and your per-item approval on the grant card, the AI may type into that tab on your behalf. Grants bind to the current process and expire when it exits or the tab closes; you can revoke anytime. Your regular shell prompt never accepts AI input.

**How do `/agents`, `@` and `/` commands work?**

Type `/` for the command menu: `/agents` binds running agents, `/new` starts a conversation, `/sessions` shows read-only history, `/memory stats` shows memory status, `/compact` compacts context, and `/help` shows help. Type `@` to select an agent by name, project or directory. Once bound, non-built-in `/commands` are forwarded after target identity and state revalidation, without consuming an LLM turn.

**What does the Agents management center do?**

Settings → Coding Agents installs, updates, uninstalls and configures agents, with custom command names and controlled config optimization. Models live in Settings → Models; enable the gateway, then turn on Attach via gateway and choose a default model on the agent's detail page. Managed keys are snapshotted before takeover and restored according to ownership and current values, without putting real endpoints or credentials into agent configs. You can also ask "install Claude Code for me" in chat.

---

## IM Channels (Feishu / Slack / WeCom)

**Can I use kaya from Feishu / Slack?**

Yes (Feishu since v2.1.0, Slack since v2.3.6). Settings → IM Channels: for Feishu the recommended path is "Connect by scanning a QR code", which completes the setup in one step; for Slack, create an app at api.slack.com (enable Socket Mode, subscribe to message events), then paste the two tokens in. Once running, message the bot in the IM and your local kaya processes it and streams the reply — direct long-connection, no public IP needed. The WeCom entry is currently greyed out ("coming soon"; existing configurations are kept and will return in a future version).

**Can strangers message the bot?**

No. The admission allowlist rejects everything from outside the list by default, and the channel won't start with an empty list. IM messages go through the same five-gate privacy pipeline as desktop chat, and platform credentials live only in the system keychain; Approve / Deny cards in IM are mutually recognized with the desktop grant cards.

---

## Remote Access (browser)

**Can I use kaya when I'm away from my computer?**

Yes (since v2.3.2). Turn on the switch in Settings → Remote Access (sign-in to your yunhou account and an active subscription required), then sign in on the yunhou website and open the "Remote Control" page to view and operate this machine's kaya from a browser — conversations, running-agent status, and approval prompts stay in real-time sync with the desktop.

**Is remote access data safe?**

The feature is off by default and explicitly opt-in; turning it off disconnects immediately. Conversation content transits the yunhou server **in memory only and is never persisted**; **API keys never transit the relay** — this machine still connects to model providers directly; everything is TLS-encrypted. Remote turns go through the same five gates as the desktop, and write operations share the same grant state as the desktop (mutually recognized).

---

## Cross-Device Coordination

**Can an agent on another computer receive work?**

Yes. Same-account devices reuse the remote-access relay. The executor must enable coordination, allow the source device and select the permitted agents; terminal writes still require authorization. The coordination master switch gates execution, not same-account discovery and status traffic, which can flow once remote access is enabled.

**Can I inspect remote output and answer confirmation menus?**

When both versions support it and the executor allows conversation relay, you can read bounded, scrubbed screen/report output on demand and answer the current menu with 1–9. Each single-use menu ticket is bound to the current instance. Chat relay and ToDo List dispatch are independent flows with no relaxation of their permissions. See [Cross-Device Coordination](docs/PRODUCT.en.md#cross-device-coordination).

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

Settings → Logs & Debug → "Open log folder"; logs live under `~/.kaya/logs/` and do not contain conversation content. Gateway privacy records are a different store that can contain sensitive originals, not diagnostic logs.

**The built-in model reports "Request Rejected by Upstream (502)" — what do I do?**

The upstream model service rejected the request (common causes: context too long / content review / insufficient quota). kaya first **auto-compacts earlier conversation history and retries once** (this only affects the AI's context — your visible chat history is unchanged); if it still fails, the error card shows guidance for the specific cause plus a one-click "New session" button to start a fresh conversation (your terminal tabs and running agents are unaffected).

**Anything else?**

Open an [Issue](../../issues) in this repository.
