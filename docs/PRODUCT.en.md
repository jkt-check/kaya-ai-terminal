# kaya User Guide

> **The AI terminal built for AI developers**

> [中文版](PRODUCT.md)

**Contents**

- [Welcome](#welcome-to-kaya) · [What kaya Can Do](#what-kaya-can-do) · [Privacy and Security Boundaries](#privacy-and-security-boundaries)
- [Install kaya](#install-kaya) · [First-time Setup](#first-time-setup) · [Model Library and Providers](#model-library-and-providers) · [Model Gateway](#model-gateway)
- [Your First Conversation](#your-first-conversation) · [Slash Commands](#slash-commands-in-chat) · [Task List](#task-list) · [Cross-Agent ToDo List](#cross-agent-todo-list-agent-handoff)
- [Terminal and Tabs](#terminal-and-tabs) · [Let the AI Read a File](#let-the-ai-read-a-file) · [Web Search](#web-search)
- [Use kaya via Feishu / Slack / WeCom](#use-kaya-via-feishu--slack--wecom) · [Remote Access](#remote-access-cloud-relay) · [Cross-Device Coordination](#cross-device-coordination)
- [Thinking Mode](#thinking-mode) · [Sessions and History](#sessions-and-history) · [Memory](#memory) · [Scheduled Tasks](#scheduled-tasks) · [Audit Records](#audit-records)
- [Settings Reference](#settings-reference) · [Keyboard Shortcuts](#keyboard-shortcuts) · [FAQ](#faq) · [Troubleshooting](#troubleshooting)

## Welcome to kaya

kaya is a macOS / Windows desktop app positioned as **the AI terminal built for AI developers**: a full terminal workspace with model management, coding-agent management, a security gateway and an AI brain. Open it and you will see two panels: the AI brain's chat panel on the left, and a real terminal on the right, just like the one you use every day (zsh on macOS, PowerShell on Windows). The AI can read scrubbed terminal output to explain commands, inspect the environment and follow agent progress; its tools execute in a separate subprocess, never by typing into your regular shell prompt.

On the model side there is a unified **model library** (two levels: provider → model, with a 200+ provider catalog) and an optional **local model gateway (security gateway)**: configure once and use everywhere, real keys never land in any agent's config file, and sensitive content is checked, blocked or scrubbed according to policy before sending, with supported replacements restored when possible.

Every terminal tab is an independent shell session: you can run Claude Code, Kimi Code, OpenCode, Codex and other coding agents in different tabs at the same time — they run **truly in parallel**. Type `@` in the chat input to summon the list of running agents; whoever you @ becomes the collaboration target of the current conversation — you assign the work, it does the work, and every confirmation and decision stays with you. Cross-agent collaboration follows naturally: bring one agent's conclusion into the chat, then @ another agent to carry it forward — you direct them all from the chat box.

kaya's goal is to give you AI assistance and multi-agent collaboration **without leaving your real shell**: you keep your shell aliases, your scripts, your familiar shortcuts, and the AI does not trap you in a sandbox.

> Note: the AI does not take over your entire terminal. By default it only reads your terminal output; only when you **turn on the coding-mode toggle in the top bar**, explicitly allow it on the grant card in the chat, and that tab is running a supported AI CLI (Claude Code / Kimi Code / OpenCode / Codex) — can the AI type into that tab on your behalf. The coding-mode toggle is the master gate for "driving": while it is off, the chat side does not have the ability to send input to terminal AIs at all (the tool is not even registered); once on, each tab still requires its own grant-card approval. The grant is bound to the current process and expires when the process exits; you can revoke it anytime via the 🤖 badge on the tab. Your regular shell prompt never accepts AI input. Likewise, when the AI helps you edit a coding agent's config file (e.g. Claude Code's settings.json), every edit lands only after you explicitly approve it on a grant card, and the grant is scoped to the current conversation.

---

## What kaya Can Do

kaya highlights **four core capabilities**, built on a **full terminal**.

| Core capability | Existing product capabilities |
|---|---|
| **Models and providers** | Model library and provider settings, model discovery and metadata, your own API keys, provider OAuth, custom compatible endpoints and local Ollama models; credential and protocol coverage is detailed in the relevant chapters |
| **Coding-agent management** | Install, configure and update Claude Code / Codex / OpenCode / Kimi Code, bind models, and inspect activity and terminal output |
| **Security gateway** | The "Model Gateway" described below, optional and off by default; only explicitly connected coding-agent model requests are checked, blocked or scrubbed according to policy, with supported replacements restored when possible and protection records available |
| **AI brain** | Use terminal context to explain commands, errors and agent plans, read progress, and summarize results and next steps; with coding mode and per-tab approval, send instructions to supported agents and hand off tasks, context and conclusions between projects |

**Full terminal**: macOS / Windows support, zsh / PowerShell, your shell profile, aliases, scripts and command-line tools, independent PTYs, multiple tabs and panes, and focus mode. Your own terminal commands are not limited by the AI tool allowlist.

**Other existing capabilities remain part of the product**: local multi-agent collaboration, the cross-agent ToDo List, cross-device coordination, IM channels, browser remote access, memory, scheduled tasks, web search, image input, session history, usage statistics, privacy audit, config optimization and auto-update all remain available. The core grouping only sets the presentation focus; it changes no support status, subscription requirement or authorization boundary. Non-core does not mean experimental. Privacy First, speed and simplicity remain product-wide principles.

The complete capabilities below remain organized into five usage groups, not another ranking of core capabilities.

**1. Coding-agent management and collaboration**

- **Parallel**: run Claude Code / Kimi Code / OpenCode / Codex in different tabs at once — one independent PTY per tab, truly parallel
- **Binding and collaboration**: type `@` (or use `/agents`) to bind any running agent as this session's forward target and assign work or follow up; one agent's conclusions can be relayed through the chat to another agent
- **Watching and driving**: turn on coding mode and kaya watches the agents in your tabs — reminding you when one waits for a decision, reading results and summarizing progress when one goes idle; with your per-tab authorization it can type into the agent on your behalf
- **Handoff and dispatch**: write follow-up work into the **cross-agent ToDo List**; once the target agent goes idle kaya asks whether to run it, dispatches only on your yes, and verifies the result against evidence
- **Remote instances**: agents running on your other device (same account) appear in the `@` menu and sidebar with a device badge — you can dispatch work and answer grant cards across machines
- **Install and manage**: the Agents management center installs / updates agents in the background, with custom command names, protocol-based model attachment, and config optimization

**2. Models and providers**

- **Unified model library**: maintain a two-level **provider → model** structure in Settings → Models. One provider carries optional **Chat / Anthropic / OpenAI** endpoints, one credential, and a model list; configure once and the chat, the coding agents, and the audit judge all share it
- **200+ provider catalog**: models.dev distillation plus a built-in overlay — picking a provider pre-fills endpoints and the model list. You can also add a custom OpenAI-compatible endpoint, and keep multiple accounts of the same provider
- **Two auth modes**: API key or OAuth sign-in; Kimi Coding Plan uses a device code, while **OpenAI ChatGPT subscriptions use browser Authorization Code + PKCE**. Tokens and the ChatGPT account identity live only in the system keychain; ChatGPT subscriptions serve coding agents through the gateway, not kaya's own Chat or audit AI judge
- **Online model discovery**: saving a key or pressing Refresh fetches that provider's model list once; on failure it falls back to the catalog snapshot labeled "may be outdated", and manually added rows are never touched
- **Model metadata**: per-model context window, max output, vision tri-state, thinking tri-state plus thinking efforts (minimal / low / medium / high / xhigh / max), and video / audio input — projected into each agent's native config according to the capabilities that agent supports
- **Model gateway (optional, privacy hardening)**: off by default; once enabled and explicitly connected, the relevant coding-agent model requests go through a local loopback gateway for policy-based checks, blocking or scrubbing, with supported replacements restored when possible. Real API keys / OAuth tokens live only in the keychain and gateway memory, **never in any agent's config file**. See the dedicated **Model Gateway** chapter below
- **Config optimization**: the AI reads scrubbed config plus bound-model metadata and proposes sourced suggestions; applying is one-time authorized, read-back verified, and auto-rolled-back on failure
- **Usage statistics**: token / request accounting shared by chat and the gateway, grouped by model / agent, bucketed by day / week / month (zero content)
- **Cloud Token usage (optional)**: a separate upload card in Settings → Account, off by default and enabled only after confirming the `kaya-usage-upload-v2` consent disclosure; it uploads only UTC-aggregated usage buckets and the device-registration fields (random device id, OS family, kaya version, consent version), never conversation or tool content

**3. Privacy and auditability**

- **Five gates**: user-input guard → policy gate → safety gate → isolated execution → output scrub, all on your machine
- **No file bodies by default**: only files you put into `~/.kaya/` can be read
- **Gateway scrubbing + privacy statistics**: per-hit tokenization / warning / restoration details stay local and can be filtered, searched, and cleared
- **Audit replay**: per-session JSONL for sources wired into auditing, plus three-pass evaluation (rule replay, anomaly heuristics, AI judge)
- **Local-only memory**: the Agent stores, reinforces, generalizes and archives memories automatically by evidence class — no review queue; you can still review, edit, roll back or delete entries afterwards

**4. Collaboration channels**

- **IM channels**: Feishu / Slack (WeCom entry still pending) over long-lived connections — no public IP needed; same privacy pipeline, grant cards answerable in IM, optional desktop-to-IM mirroring
- **Remote access**: view and operate this machine from a browser; the relay forwards in memory without persisting
- **Cross-device coordination**: two devices under one account dispatch work to each other; executor admission is fail-closed, grants are answerable on both ends with authority on the executor
- **Scheduled tasks**: fire chat turns on an interval or at a fixed time each day

**5. AI brain and everyday development assistant**

- **Code explanation**: paste an error, a shell script, or a regex and ask the AI what it does
- **Environment checks**: inspect your system info, disk usage, running processes, network ports, and so on
- **File queries**: list directory contents, count file sizes, find files by name pattern
- **Command construction**: let the AI compose `find` / `sed` / `sort` pipelines and run allowlisted commands in an isolated subprocess, returning the output
- **Log and error interpretation**: have the AI explain a block of error output and suggest possible causes
- **A real terminal**: multiple tabs, split panes, focus mode, with a PTY that loads your real `.zshrc` / PowerShell profile

kaya does **not** do, and does not try to do, the following:

- Having kaya's own AI tools directly rewrite source code on your computer (running a coding agent in a tab to write code is another matter — that is the agent's own capability, always visible and interruptible in your terminal)
- Editing a coding agent's config file without your explicit authorization (every config-optimization write requires you to tick items and approve a card)
- Running destructive operations without your confirmation (for example, deleting files)
- Having its own AI run interactive programs in your terminal (such as `vim` or REPLs)
- Accessing the body of any file outside `~/.kaya/` without your permission
- Answering a coding agent's own confirmation for you without your authorization — that is the agent's own safety gate. kaya never answers on its own; once you grant per-item authorization (coding mode), input on your behalf is guarded by multi-check revalidation, and the decision is always made by you

On "who executes": kaya's AI runs allowlisted commands in an **isolated subprocess** and shows you stdout / stderr directly — you don't need to copy them into your terminal and run them again; destructive, interactive, and system-modifying commands are blocked before execution. Commands **you** type in the terminal pass through no gate at all — that area is yours. Both share the same principle: the AI assists; decisions and authorizations stay with you.

---

## Privacy and Security Boundaries

Privacy is the **first priority** of kaya. Below is the boundary you can expect by default.

**What the AI Cannot See by Default**

- The **contents** of any file on your computer: read commands like `cat`, `head`, `grep`, `less` are blocked by default
- Your system environment variables: `env` and `printenv` are blocked by default

**What the AI Can Query by Default**

- **Metadata**: directory listings, file sizes, modification times, current user, system version, and so on
- Terminal output (so you can ask the AI to explain an error); reads pass through scrubbing
- **Where running agents live and what they have been doing** (which tab, which directory, recent activity), plus the tail of their current screen
- Your task list (read-only plus controlled creation) and cross-agent ToDo List (the AI can view it and create handoff tasks, but you decide when they run and state transitions are validated and applied by kaya)
- Scheduled tasks (create / enable / disable / delete — the same data as the Settings page)
- **Memories in effect**: core rules / facts and the memory index (adjudicated automatically, manageable after the fact); archived cold storage and superseded entries never enter default injection or retrieval
- Text and images you actively paste into the chat

**What the AI cannot get by default**

- The **body** of any file (unless it lives under `~/.kaya/`)
- System environment variables
- Archived cold-storage and superseded memory entries: excluded from default injection and retrieval; they become retrievable when the model explicitly searches with `include_archived` (archived **rules** are the exception — rules never enter the retrieval pool, so they can only be found in Settings → Memory)
- Cross-device execution, on-demand screen reads and conversation relay without executor admission and the relevant permissions; discovery and status summaries have separate boundaries described under Cross-Device Coordination
- Raw API keys / OAuth tokens (the frontend structurally never receives them)

Commands you type into the terminal do not pass through the AI tools' policy or safety checks. Terminal output can still be read by the AI's read-only tools and sent, after scrubbing, to the current model. Not intercepting your commands does not mean terminal output never enters model context.

**The `~/.kaya/` Trusted Directory**

If you need the AI to read the contents of a file, place the file under `~/.kaya/` and tell the AI the path. This is the **only** location where the AI can read file bodies. See [Let the AI Read a File](#let-the-ai-read-a-file) for details.

The trusted root is not an unrestricted read/write area: credentials, gateway data, the model library, config snapshots and upload state have additional write-denial or sensitive-read rules. Desktop, IM and cross-device tools also have different read scopes. See the [privacy design](PRIVACY.en.md) for the full boundaries.

**Scrubbing Is a Defense, Not an Absolute Guarantee**

When you input text, when the AI reads a file, and when tool results come back, kaya automatically recognizes and replaces common keys, PEM private keys, ID numbers, phone numbers, credit card numbers, and other sensitive patterns. The built-in sensitive-data detection covers 30+ rule families across common scenarios. But kaya does **not** promise to recognize **every** possible secret — do not paste truly important keys into any LLM application.

**What Cloud Providers Receive**

When you send a message in the chat or let the AI call a tool, the scrubbed chat text and scrubbed tool results are sent to the **model source you picked in the top-bar model switcher** (Built-in model / My Models):

- **Built-in model (default, yunhou `/chat` proxy)**: scrubbed content is sent over your login token (JWT) to the **yunhou first-party server**, which answers on your behalf with yunhou's DeepSeek key. Before sending, the kaya client trims to the server's limits (message count / per-message bytes / total bytes — the oldest history is dropped first). Your login token (refresh_token) lives only in the system keychain (macOS Keychain / Windows Credential Manager). The "Built-in model" entry in the top-bar model switcher expands into the server-provided model catalog: the default is "Follow default" (no pinned model id — tracks the server-side default as it changes), or you can pick a specific model from the catalog; when the catalog is unavailable (signed out / no access / not enabled), it falls back to a single static row.
- **OpenAI compatible (your own key)**: scrubbed content goes to the LLM provider you chose (DeepSeek / OpenAI / OpenRouter / etc.), authenticated with your API key.
- **Ollama (local)**: inference and conversation / tool content stay on the machine (see below).

kaya does not send your content to unlisted or unauthorized destinations. Beyond the model endpoints above, explicitly enabled **Tavily web search**, **IM channels and mirroring**, and the **yunhou relay used by remote access / cross-device coordination** also transmit scrubbed content as required by those features; each path is listed in its corresponding chapter. Local UI state, sessions, audit logs, and application logs all stay on your computer and are not automatically synced to the cloud; the signed-in usage heartbeat and explicitly consented cloud Token usage upload are separate channels that carry no conversation or tool content.

**How Your API Key and OAuth Credentials Are Used**

The API key you enter in Settings is **stored only in your local system keychain**, and is sent only to the model endpoint you configured — it is never uploaded to Yunhou and never written to audit logs. kaya does **not** send your API key to any other party. kaya does **not** promise "never uploaded" — any network request needs authentication, and authentication needs the key, that is the basic mechanism of network requests.

On macOS, all kaya secrets (login token, BYOK keys, OAuth tokens, Tavily key, IM channel credentials) converge into a single unified keychain item, so a normal launch asks for authorization at most once; the first launch migrates old entries and may still touch them once. Rolling back to an older app version means it cannot read the new storage and will report "not configured" — the data is not lost, but credentials must be re-entered. Windows keeps its existing Credential Manager layout.

Provider OAuth tokens (Kimi Coding Plan) also live only in the keychain: the frontend structurally never receives them (the IPC payload has no token field), and the real bearer is used only inside the Rust request path and gateway memory. Every enabled endpoint of an OAuth provider must match the **same host** declared by the catalog, otherwise the credential is not sent — even if someone edits the stored base_url. Signing out clears only local credentials and does not notify the server.

**Coding-agent traffic (optional): the model gateway**

The chat path and the coding-agent path are **independent** outbound routes:

- **Chat**: the model you picked uses its native API (built-in model / my models), passing through the five gates. kaya never reroutes chat through the gateway.
- **Coding agents**: by default each agent connects to its provider on its own (if you never attach it, kaya does not touch it). Once you turn on **Attach via gateway** on an agent's detail page, its model traffic goes through a local loopback gateway — a hop over `127.0.0.1`, reversibly tokenizing outbound content and restoring inbound responses. Real API keys / OAuth bearers live only in the system keychain and gateway process memory and **never land in any agent's config file**.

The gateway scrub level is independent: **strict** by default (named secrets only; ordinary PII is left unchanged), with balanced / aggressive available. On the chat side user input is always aggressive and tool output defaults to aggressive. Per-hit details (which rule, which original text, which token, how many restorations) land locally in `~/.kaya/gateway/privacy-events.jsonl` and are visible, filterable, and clearable under **Settings → Gateway → Privacy statistics** — local only, masked by default, never fed into LLM context or diagnostic exports, and a different store from the **chat audit trail** (which keeps scrubbed text); the only **content-free** audit event is the gateway scrub event `gateway-scrub`. Full details are in the **Model Gateway** chapter below.

**Offline Inference: Ollama**

If you choose Ollama as the provider, all inference happens **entirely on your machine**, and inference plus conversation / tool content is **not sent out**. The signed-in usage heartbeat and explicitly consented cloud Token usage upload are separate channels, independent of the provider choice. This is suitable for the most privacy-sensitive scenarios. Local endpoints (Ollama / LM Studio / vLLM) may run without a key.

**Five Layers of Protection, in Brief**

In short, everything from your input to the AI's output goes through:

1. **Input guard**: pasted keys and long text are scrubbed and truncated
2. **Command policy**: executable commands are restricted by an allowlist; unknown commands are denied by default
3. **Safety check**: destructive, interactive, and system-modifying commands are blocked
4. **Isolated execution**: AI tool calls run in independent subprocesses and never touch your terminal
5. **Output scrub**: tool output is scrubbed again before entering the chat

---

## Install kaya

> kaya ships in two build variants: the **China build (cn)** at [yunhouai.com](https://www.yunhouai.com/download) (WeChat sign-in, WeChat Pay), and the **international build (intl)** at [yunhou.ai](https://yunhou.ai/download) (GitHub sign-in, PayPal). Download installers from the corresponding official website. This repository publishes documentation only, not source code or installers.

**macOS: Download the DMG**

1. Visit the [official international download page](https://yunhou.ai/download) (or the [China download page](https://www.yunhouai.com/download)) and download `kaya-intl-vX.Y.Z-macos-universal.dmg` (China build: `kaya-vX.Y.Z-macos-universal.dmg`; universal for Apple Silicon and Intel; requires macOS 12 or later)
2. Double-click the DMG to mount it; a window showing `kaya.app` will appear
3. Drag `kaya.app` into the `/Applications` folder
4. Open kaya from Launchpad or `/Applications`

**Windows: Download the installer (x64)**

1. Visit the [official international download page](https://yunhou.ai/download) (or the [China download page](https://www.yunhouai.com/download)) and download `kaya-intl-vX.Y.Z-windows-x64.exe` (China build: `kaya-vX.Y.Z-windows-x64.exe`; NSIS installer; an `.msi` of the same version is also available)
2. Run the installer and follow the wizard
3. Open kaya from the Start menu

**First Launch (macOS)**

If macOS shows "cannot be opened because Apple cannot check it for malicious software":

1. Find `/Applications/kaya.app` in Finder
2. Allow the verified kaya app in **Privacy & Security** in System Settings
3. Open kaya again

Do not disable Gatekeeper, and do not run privilege-escalation commands from unknown sources.

> The official installers are the recommended way (DMG on macOS, exe / msi on Windows). Pre-compiled CLI releases are currently paused; advanced users who need the CLI should refer to the separate CLI documentation.

---

## First-time Setup

After opening kaya for the first time, do the following.

**Option A (default, no API key): built-in model + WeChat sign-in**

By default kaya uses the **built-in model** (the yunhou `/chat` proxy — the server answers with yunhou's DeepSeek key), so you don't need to apply for an API key. You only need to sign in to your yunhou account:

1. Click **⚙ Settings**, or click **Sign in with WeChat** in the chat window (the account icon / sign-in button in the top bar)
2. Your system browser opens a WeChat scan page → scan → you're redirected back to kaya (the token is written to the system keychain: macOS Keychain / Windows Credential Manager)
3. Return to the chat window and ask away. Subscription / trial status is under **⚙ Settings → Account**

> On the international build (intl), sign-in is via **GitHub** (also completed in the system browser with an automatic redirect back to kaya), and subscription payment goes through the **PayPal** checkout (website: yunhou.ai); the rest of the flow is the same as the China build.

> **Two-tier feature gating**: sign-in allows free bring-your-own-key (BYOK) chat; an active paid subscription unlocks built-in models, coding mode, IM channels and remote access. The core-capability grouping does not change these requirements.

**Option B (your own API key): OpenAI compatible**

If you want to use your own LLM key:

1. Click the **model switcher in the top bar** and switch to **My Models** (you can switch back to **Built-in model** anytime)
2. Open **⚙ Settings → Models** and click **+ Add Provider**:
   - **Provider**: pick from a catalog of 200+ providers (models.dev distillation + built-in overlay, including OpenCode Go), or choose **Custom** (OpenAI compatible)
   - **Authentication**: an API key from your provider's website, stored in the system keychain; Kimi Coding Plan uses OAuth device code, while **OpenAI ChatGPT subscriptions use browser OAuth + PKCE** (the token and account identity stay in the keychain, sign-out is available, and use is limited to coding agents through the gateway). OpenAI API / Anthropic entries currently offer API keys only
   - **Endpoints + model list**: endpoints and the model list are pre-filled from the catalog and usually need no changes; a single provider can carry **Chat / Anthropic / OpenAI** endpoints at once, and multiple models of the same protocol family share one provider and one key — configure once, use everywhere
3. To use a model in a coding agent, first turn on **Enable gateway** in **Settings → Gateway**, then open **Settings → Coding Agents** → that agent's detail page, and in the **Model Gateway** card pick the provider's model as the **default model** and turn on **Attach via gateway** (see **Model Gateway** below)
4. Return to the chat and ask away

Once configured, kaya remembers your settings. You do not need to reconfigure next time.

> Models are maintained in one place — **Settings → Models** (a two-level provider → model structure; a single provider can carry Chat / Anthropic / OpenAI endpoints plus a model list), so configuring once makes it usable everywhere; **Settings → Coding Agents only picks** — in the agent detail page's **Model Gateway** card, turn on **Attach via gateway** and pick a default model. API keys all live in the system keychain (macOS Keychain / Windows Credential Manager).

---

## Model Library and Providers

**One thing, one place**: the model library lives in **⚙ Settings → Models**, the **only** place to edit models and credentials. The top-bar model switcher only chooses "which model chat uses now", and **Settings → Coding Agents** only decides "which library model an agent uses, and whether through the gateway".

**Two levels: provider → model**

A **provider** is one credential plus optional three-protocol endpoints (Chat / Anthropic / OpenAI) plus a model list. Models under a provider share one key, so "one API key configured once works everywhere"; each model carries its own enabled state, context window, capabilities, and thinking efforts.

**Adding a provider**

1. Click **+ Add Provider** and pick from the **200+ provider catalog** (models.dev distillation plus a built-in overlay, including OpenCode Go); endpoints and the model list are pre-filled. The catalog is compiled in, so browsing it does **not** go online.
2. For providers not in the catalog choose **Custom** and fill in its OpenAI-compatible endpoint (self-hosted vLLM / LM Studio / a local gateway all go this way).
3. **Multiple accounts of the same provider are allowed** (same endpoint, different keys), each named and attached to different coding agents.

**Authentication**

- **API Key**: obtained from the provider, stored in the system keychain; the library file holds **no** keys.
- **OAuth sign-in**: Kimi Coding Plan uses a device code; **OpenAI ChatGPT subscriptions use browser Authorization Code + PKCE** (click **Sign in**, approve in your system browser, then check the signed-in status; re-login and sign-out are available). Tokens and the ChatGPT account identity stay in the keychain, the frontend never sees them, and signing out clears only local credentials. OpenAI API / Anthropic entries have no startable OAuth flow and fail closed. ChatGPT subscriptions serve only the four coding agents through gateway Responses adaptation, not kaya's own Chat or audit AI judge. OpenAI has been verified on macOS for a first Codex request, long-history continuation and built-in bare model names under global takeover; Kimi Coding Plan real-device verification remains pending.
- **Local endpoints need no key**: `localhost` / `127.0.0.1` endpoints may run without a key (Ollama / LM Studio / vLLM).

**Model list and online discovery**

The model list is not a dead list:

- Saving a non-empty API key fetches that provider's model list once; the detail page also has a **Refresh** button at any time (with the last fetch time shown).
- Opening a provider that is in the library but was never fetched also triggers one fetch; OAuth providers skip it when signed out / expired, and fetch once after a successful login.
- **Only your actions trigger it — no background polling**; requests go only to that provider's own endpoint, with the key in the auth header only.
- **ChatGPT subscription exception**: the model catalog is compiled in; online `/models` requests and connection probes are not sent.
- On failure the reason is shown (auth / network / timeout / response too large / no usable models…). Providers with a catalog source fall back to the catalog snapshot labeled "may be outdated" — **manually added rows are untouched**, and a transient network failure never deletes models you already discovered.

**What each model can declare**

| Item | Description |
|---|---|
| Enabled | Whether it takes part in projection and pickers; disabled models do not appear in an agent's model surface |
| Context window | Exact token count, or shorthand like `256K` / `1.5M` (1024-based); commits on blur or Enter, rejects values under 1K; empty = unknown (defaults apply) |
| Max output | Per-response generation cap, used by agents that need the field; conservative default instead of 0 when unknown |
| Image input (vision) | Auto / Supported / Not supported. Auto resolves via provider-catalog distillation (`modalities.input`) → LiteLLM snapshot. When neither knows the model, **the chat side still sends after a warning**; the projection to coding agents treats it as not supported (omitting `image_in` only blocks images, while claiming it wrongly gets a 400 from upstream) |
| Thinking | Auto / Supported / Not supported. **An explicit declaration overrides the catalog**, including an explicit "not supported" |
| Thinking efforts | Multi-select of minimal / low / medium / high / xhigh / max plus a default effort; explicit efforts project into each agent's native keys (kimi `support_efforts` / `default_effort`, codex `model_reasoning_effort`, opencode `options.reasoningEffort`, claude's supported capabilities + effort level) |
| Capabilities | Video / audio input are checkable; image input derives from the effective vision value and tool use is always on (both read-only). That row is consumed by kimi-code today |

**Clear declaration** removes the thinking and effort declarations together (removing only the boolean is not enough), returning the model to automatic inference.

**How changes propagate**

- Editing a provider (endpoints / model list / key / capabilities) makes attached agents **follow automatically** — no second action.
- Clearing an API key invalidates the gateway credential too; agents depending on it drop out of gateway form and fall back to their own configuration.
- Deleting a provider drops kaya's configuration from attached agents and **deletes that key** (the confirmation dialog says so).
- **Agents that were never attached are never touched.**

**Status and diagnostics**

- Each provider row in the left column has a status dot: built-in models check sign-in and subscription; BYOK providers check credential and endpoint readiness (API key set, or OAuth signed in) and show **which agents are using it** (only truly gateway-attached ones count).
- Supported services (DeepSeek / Moonshot (`api.moonshot.cn` / `api.moonshot.ai`) / SiliconFlow / OpenRouter, etc.) show that account's **balance** (a read-only lookup: the key is read locally and sent only to that provider for authentication, never relayed by kaya, never logged, never in audit; unrecognized providers show nothing).
- **Test connection**: any HTTP response counts as "endpoint reachable" — 2xx/3xx is normal, 401/403 means "reachable but the credential may be invalid", and timeouts / transport failures are reported separately. It does **not** verify that a model works, and it does not write to the library.
- **Connection status**: per agent, shows "Connected" or "Not connected + reason" (missing credential / missing or disabled endpoint / protocol incompatible / model disabled). While the gateway is not running it shows "The gateway is not running, so the connection status is unknown (credentials were not checked)" instead of pretending it checked.

---

## Model Gateway

The model gateway is the core **security gateway** capability: optional and off by default, processing only explicitly connected coding-agent model requests, not chat or all system traffic. It checks, blocks or scrubs according to its own policy, restores supported replacements when possible, and provides local protection records. Available models depend on each agent's protocol and credential coverage.

**The two problems it solves**

1. **No more keys in agent config files.** Without the gateway, making Claude Code use your third-party key means writing the real key into its config — a plaintext copy that gets cloud-synced and read by other tools. With the gateway, the agent config holds only a per-agent placeholder token (`kayagw_…`) and an alias; the real key lives only in your keychain and the gateway process memory.
2. **Reduce verbatim sensitive-content egress according to policy.** Matches selected for rewriting become **reversible tokens**, and supported response replacements are restored when possible. Tokens aim to preserve the shape of emails, phone numbers, national IDs and card numbers. Default strict mode does not automatically rewrite ordinary PII; detection and restoration are not absolute guarantees.

**Three steps to turn it on**

1. Configure providers and models in **⚙ Settings → Models** (see the previous chapter).
2. Turn on **Enable gateway** in **⚙ Settings → Gateway**. The status line shows "running · port" when ready; the gateway only listens on `127.0.0.1` and is not exposed.
3. Open an agent's detail page under **⚙ Settings → Coding Agents** → the **Model Gateway** card: turn on **Attach via gateway** and pick a **default model**.

New bindings attach via the gateway by default; if the gateway is not enabled yet the toggle shows "Will attach via the gateway once it is enabled" instead of silently doing nothing.

**Hard boundary: only "gateway form" or "revoked"**

This is the key to understanding the gateway:

- What lands in an agent's config is always **loopback address + placeholder token + model alias `kaya/<alias>` + a friendly display name** (e.g. "Deepseek (deepseek-flash)").
- **Real base URLs and real keys never land in agent config.** kaya also never hands library models to an agent as "direct" connections — there is no third form.
- **Placeholder credentials cannot authenticate upstream requests**: after real credentials are injected, `kayagw_` in authentication headers blocks sending; provider-declared custom headers also reject `kaya/` aliases. Content requests translate the top-level model alias into the real model. Metadata passthrough can preserve a resolvable alias, so this is not a promise to inspect and remove aliases from every request body. See privacy design §7.11.
- Turning the toggle off means **revoke** — strip what kaya wrote and restore from the pre-takeover snapshot; it is never "switch back to direct".

**Full catalog and protocol-based routing**

- The gateway's catalog is the **entire library** (provider × model), expanded into **Anthropic / OpenAI Chat / OpenAI Responses** protocol slices; `/v1/models` is answered by the gateway itself, so an agent's model picker can see every usable model in the library, not just the bound one.
- Each agent's protocol family is fixed: Claude Code uses Anthropic; **Kimi Code / OpenCode use Chat-compatible**; Codex uses Responses.
- **OpenAI ChatGPT subscriptions** use the official Codex Responses upstream: Codex uses native Responses, while Kimi Code / OpenCode Chat and Claude Code Anthropic Messages are reverse-adapted by the gateway into Responses, so all four agents can share one subscription credential. A missing account identity fails closed.
- When a provider only has a Chat endpoint, Codex automatically gets a **Responses→Chat adaptation** slice (marked "Responses→Chat" in the catalog), and upstream 4xx/5xx error shapes are translated back into Responses shapes.
- **Fail-closed**: a request whose model name is not a resolvable `kaya/<alias>` (bare upstream name, malformed name, cross-family alias) gets **HTTP 400** with a reason, a hint, and a docs link — it is **never silently forwarded**. The sole exception is an explicitly globally taken-over Codex bound to ChatGPT Responses, where Codex may select its built-in bare model names; ordinary profiles, other providers, and third-party tools still reject bare names. `/health` / `/healthz` answer locally without a token, for liveness checks.

**Takeover and restore: your original config is safe**

Once explicitly attached, kaya takes over the agent's **model surface** — provider block, kaya model entries, default-model pointer, picker / catalog list, and opencode's provider gate. It does **not** touch permissions, hooks, MCP, plugins, or the keys config optimization manages.

- Before takeover it snapshots those keys **one by one** into `~/.kaya/agent-config-snapshots/` (0o600).
- Non-kaya model / provider entries that existed at takeover are snapshotted, then removed or hidden; entries you add afterwards are kept.
- Revoke merges back ("fill only what is missing, current entries of the same name win") and never overwrites entries you defined later.
- If the snapshot cannot be taken (read failure / lock busy), kaya **does not delete or overwrite** user values it cannot restore — it writes only into its own namespace and leaves a trace. Better not to take over than to delete irreversibly.
- **Drift**: if the config file is effectively changed outside kaya, the detail page shows "config edited outside kaya", kaya does not overwrite it, and **Re-attach** rewrites the expected form.
- **Codex is isolated by default**: kaya writes only its own profile (`codex --profile kaya`), affecting Codex launched from kaya or explicitly selecting that profile. **Global takeover codex** writes the official base config instead, affecting plain codex and sessions launched by the ChatGPT desktop app — it asks for a second confirmation, and stopping the gateway / quitting / crashing all restore from the snapshot.
- **Show bound model only**: when on, that agent's model picker shows only the bound model; when off, the whole usable catalog.
- **Revoke is idempotent**: a second revoke only finishes up and never deletes the user `base_url` / token restored by the first round.
- **Boundary (stated honestly)**: configurations taken over before the snapshot mechanism existed have no snapshot; revoke degrades to stripping only kaya's keys, and you must restore old values by hand.

**Scrubbing: an independent level**

| Level | Behavior |
|---|---|
| **strict (default)** | Named secrets only; ordinary PII is left unchanged |
| **balanced** | Named secrets and PII tokenized; high-entropy fallback **only warns** |
| **aggressive** | Most aggressive: even high-entropy strings are tokenized (more false positives on encoded data, lower restoration reliability) |

- The gateway level is **independent of chat**: chat user input is always aggressive, tool output defaults to aggressive (adjustable under General).
- Tokens try to **preserve shape**: emails, phone numbers, ID numbers (18 digits with a recomputed GB11643 checksum), card numbers (separators preserved, Luhn recomputed), JWTs, and so on.
- PEM private keys are matched **as a whole block** (BEGIN + body + END), never split.
- **Protocol-field-aware handling**: named secrets are always tokenized, while pattern PII and high-entropy content in code, tool arguments, and tool results is observed without rewriting, protecting function arguments, JSON, and `tool_call_id` values.
- **Exact media validation**: valid PNG / JPEG / GIF / WebP / PDF / WAV / MP3 binaries pass through unchanged, including standard image / PDF parts in Responses tool outputs and audio parts in Chat / Responses; `text/*` carriers are processed as text. Unsupported media, conflicting declarations, or validation failures return HTTP 400 without forwarding upstream, while malformed nesting and extra business fields still undergo privacy scanning.
- **Public documentation examples**: the complete public access key ID example in AWS documentation is not replaced as a real secret; similar values remain protected, and explicit user token mappings always take priority.
- Public protocol IDs (`call_` / `toolu_` / `tool_` / `fc_` / `resp_` / `msg_`) are exempt from the high-entropy fallback (otherwise tool-call IDs would be smeared) but **named rules still apply**.
- **User token map**: fixed replacement rules — outbound "truth" is replaced with a "token", and responses are restored as-is (for example replacing the real name `Zhang Wei` with `person-a`). Useful for "I don't want the model to see real names / internal codenames in this project".
- **Reject unknown paths** is a separate switch: by default unrecognized API paths are passed through with a warning; when on they are rejected (which can break an agent's auxiliary requests, so it is off by default).
- **Restoration is not absolute**: known tokens in a response are restored automatically, but if the model rewrites or mistypes a token, restoration cannot be guaranteed. That is an inherent trade-off of reversible tokenization, not a bug.

**Privacy statistics (per-hit details)**

Under Settings → Gateway, the top sub-tab **Privacy statistics**:

- Named rules (secrets / PII) are shown first by default; low-confidence **high-entropy** records are revealed on demand.
- Filter by category, action, agent, model route, rule type, and time window; search by original text or token; clicking a category / action label filters by it.
- Hits of the same content within one request merge and accumulate an **occurrence count**, with first / last seen times and the raw record count.
- **Details contain the matched original and the replacement token**, stored only locally (`~/.kaya/gateway/privacy-events.jsonl`, 0o600), masked by default, expanded on click, and never automatically added to LLM context or diagnostic exports.
- The view reads the last 90 days with at most 5,000 records; **Clear current view** or **Clear all records** both require confirmation.
- **Request metrics and recent events** separately show request totals, rewritten requests, upstream 4xx responses, exact restores, and media handling. Each Content request keeps only structural counts and status, never its body, tokens, or full field paths.
- **Compatibility diagnostics**: when signed reasoning is rejected locally to preserve protocol integrity, a separate content-free local event can be inspected through fixed log paths and commands. It stores structural information such as the request ID, field location, and decision, never original text, tokens, or signed content; it is neither uploaded nor included in diagnostic exports. A full clear also removes compatibility events, while a selective clear leaves them untouched; failures while clearing related logs are explicitly reported as partial failures.
- This is a separate store from the **content-free** gateway scrub event `gateway-scrub`: the scrub event records "what category of event happened", the privacy details record "what exactly was replaced". The chat audit trail is a third thing — it keeps scrubbed text.

**Upstream failure behavior**

- **Streaming**: after 120 seconds (default; configurable in the gateway config) without data the gateway aborts and sends a **terminal error frame in the agent's protocol** (Anthropic `event: error` / Chat error chunk + `[DONE]` / Responses `response.failed`) — it **never fakes completion**. The agent sees a failed request and its own retry logic takes over instead of hanging forever.
- **Non-streaming / waiting for headers**: each has a 300-second cap, timing out with a structured 504.
- Timeouts and broken streams are written to the **App diagnostic log** (Settings → Logs & Debug) with route identifiers and seconds only — **zero content**; the gateway page does not display these counters.

**What it does not do**

- It does not reroute chat traffic: chat connects directly to the model you picked.
- It does not store your request content: gateway audit is content-free, and privacy details are a separate local store.
- It is not a transparent proxy: models outside the library are rejected (HTTP 400 with guidance), never silently passed through.
- It does not take over an agent's non-model configuration (permissions / hooks / MCP / plugins / optimization keys are untouched).
- **Only one instance may hold it**: the dev and prod gateways are mutually exclusive; when occupied, write operations are rejected and name the instance holding it.

**Usage**

Chat and the gateway share the same token accounting. **Settings → Models → Usage** shows totals for the last 90 days (default window), per-model and per-agent sums, with day / week / month buckets and agent filtering. It stores only provider / model / agent identifiers and actual counts, never request bodies, and never invents usage the upstream did not return. Auxiliary calls outside the agent loop (fast decision / task distillation / memory consolidation) are excluded; entry routing itself is not counted, while the slim turn it triggers inside the agent loop is.

---

## Your First Conversation

After configuration, type this in the chat input at the bottom left:

```
Show the current system version and CPU architecture
```

Press **Enter** to send. You will see:

1. The AI first shows a collapsible reasoning block (when the current model supports and enables thinking)
2. Then a **tool card** appears, showing which tool the AI called and the arguments
3. The card status changes from **running** to **done**; expand it to see the full stdout / stderr
4. The AI summarizes the result in natural language

Replies follow the language of your current message, with explicit language requests taking priority. With an English interface, ambiguous messages default to English. Tool cards preserve the actual tool output in its original language.

**Recognize the Three Elements**

- **Reasoning block**: a collapsible gray section showing the AI's reasoning
- **Tool card**: appears whenever the AI calls a tool; shows the tool name, arguments, status, and output
- **Final answer**: the AI's natural-language conclusion

If the result is not what you want, follow up:

```
What about disk space usage? Only files owned by the current user.
```

If the AI stalls or answers poorly:

- Press **Esc** or click the **⏹** button to interrupt the current turn
- Then rephrase and try again

**Paste Images**

Press **⌘V** (**Ctrl+V** on Windows) in the chat input to paste images: kaya automatically strips metadata such as EXIF / GPS and downsamples oversized images. Up to 4 images per message, 5 MB each; png / jpeg / webp / gif are supported. Images go only to the model you picked: they are sent along with the message only when you use your own model (BYOK) and that model supports image input; under **Settings → Models**, each entry can declare **Image input: Auto / Supported / Not supported** — "Auto" resolves through catalog distillation → the LiteLLM snapshot. The builtin model cannot see images yet; sending with images attached is blocked with a notice. Images are in-memory: never written to disk, never in audit content (only mime and byte count are recorded). Sent messages carry a **Send to agent** button that stages the image under `~/.kaya/tmp-images/` (0o600) and hands the **absolute path** to a coding agent running in a terminal tab (the agent itself must support reading images). Images do not persist across sessions: after a new conversation, images in earlier messages show a "Cleared with the session" placeholder.

**Tips**

The conversation area shows kaya usage tips: a welcome card when the conversation is empty, and a one-line tip bar above the composer once messages exist, rotating every 5 minutes; tips for features you have already used are automatically deprioritized. Click **Next tip** to cycle immediately, or **Don't show again** to turn them off for good.

---

## Slash Commands in Chat

Type **`/`** in the chat input to summon the command menu (pick with the keyboard or mouse to autocomplete); `/help` lists all commands anytime. The built-in commands:

- **`/agents`**: lists the currently running companion agents (kimi / claude / opencode / codex, etc.). Click a card to bind it as this session's forward target — the binding is per-session and cleared on a new conversation
- **`/new`**: start a new conversation (same entry point as the New button in the top bar)
- **`/sessions`**: lists past sessions from the audit trail; click a card to view its read-only history
- **`/memory stats`**: show memory-store statistics and health
- **`/compact`**: compress the current conversation's context into a summary and continue from it (your visible chat history stays unchanged); memory is distilled once first, so important facts are not lost to compaction
- **`/help`**: lists every command with a description

Type **`@`** in the chat input to summon the list of currently running companion agents (keep typing to filter, just like `/agents` — name, instance ID, project label, and working directory all match, with name/project hits shown first), and each menu item's second line shows the agent's state, working directory, and current activity; pick one with the keyboard or mouse to bind it as this session's forward target, and a recognizable label (`@instance ID · project (directory)`) is completed into the input box — you can see exactly who the message will go to. Subsequent messages route to that agent through the same binding channel, no need to open `/agents` first.

Input that starts with `/` but isn't a built-in command (such as `/auto`) does not error out: if the current session has a bound agent, kaya writes the original text into that agent's terminal input line and presses Enter for you. Before writing, it fails closed on several checks (the pane is still alive, the foreground process is still the bound agent, and it is not waiting at a menu); if any check fails, nothing is written. These forwards **do not consume an LLM turn**; each forward appears as a card in the message stream and is recorded in the audit log. When you re-bind across tabs, kaya attaches a one-time context note to your next message so the AI knows what came before (the note goes to kaya's AI — it is not typed into the agent's terminal).

---

## Task List

kaya ships a lightweight task list to help you and the AI track what needs doing:

- The todo toggle in the top bar opens the **Task panel**, where you can view, check off, and manage current tasks
- You can also manage it right in the chat with commands like "Add task …" / "Update task … to …" / "Delete task …" / "Clear tasks": these are recognized and executed **locally by the client**, without involving the LLM; free-form phrasing (e.g. "mark buying milk as done") is not recognized as a command and goes to the AI as a normal message. The "extract tasks" command asks the AI to draft tasks that land in the list after your confirmation
- Tasks you don't need right now can be **archived**: archived tasks drop out of the main list and the count into a collapsed "Archived" section at the bottom of the panel, and can be restored anytime; archived tasks never appear in the list snapshot the AI sees
- On the AI side the tool is **read + controlled create**: it always sees a snapshot of the list to stay aligned; when you ask it to create a task (e.g. "add a todo: buy milk"), it can — the title lands after client-side validation and duplicate open tasks are skipped; but it cannot check off, delete, or archive tasks for you

---

## Cross-Agent ToDo List (Agent Handoff)

The task list tracks **your long-term todos**; the cross-agent ToDo List tracks **tasks to be executed by a running agent** — two independent entities that never mix:

- **What it solves**: preserving context across agents and picking the right moment to run. When one agent finishes its current work and the next step needs **a different agent**, the AI writes the task (title + handoff context) into the ToDo List; the context is stored with the entry and survives new sessions and app restarts
- **You decide when it runs (default)**: when the target agent goes idle, the AI in chat asks "run it now?" — only your explicit yes dispatches it. kaya knows each coding agent's current state best, so it picks the moment to ask; you can also click **Dispatch** on an entry anytime. Prefer less friction? Switch **Command confirmation** in Settings → Models to **Auto mode**: kaya dispatches as soon as the timing is right, without asking each time (the per-tab terminal-write grant cards are unaffected)
- **Agents finishing on their own? kaya sees it**: kaya keeps an eye on every running agent; when the target agent goes idle, it first checks queued tasks against on-screen evidence — if the agent already completed a task on its own, kaya marks the entry done and tells you instead of dispatching it again; if the agent is already working on it, kaya won't nag twice
- **Work already underway shows as Running in real time**: if the task was in fact already handed to the target agent (e.g. kaya dispatched it via a direct terminal write) before the entry was created, or the entry's agent just started working, kaya adopts it on sight — the entry flips from Pending to Running immediately and enters the acceptance loop instead of sitting in the queue; verification runs as usual once the agent goes idle. When several instances of the same agent are open, kaya can't tell which one to track, so the entry stays pending and is not adopted
- **Where it lives**: the **To Do List area** above the chat composer — collapsed by default; one click expands it to show pending entries and their states. Completed entries move into a collapsed **Done** section with completion time, conclusion, and evidence (up to 50 entries, clearable in one click). The expanded area also has **Summarize progress**, which asks the AI to read the screens and post a cross-repository progress report into the chat
- **Four states**: Pending / Running / Done / Failed. Completion no longer means "delete after acceptance" — completed entries persist in the Done section. Failed entries stay with a structured reason (execution failed / perception unclear / target exited / environment blocked); the AI explains it, and you can choose **Retry**, **Reobserve**, **Mark done**, **Redispatch**, or delete as appropriate
- **Perception closes the loop**: an idle agent does not mean the task is done. Once the agent goes idle, kaya runs a **perception turn** (observation and acceptance together) against three evidence sources: kaya's own read-only git digest (branch / uncommitted changes / recent commits, scrubbed), the target agent's activity ledger, and the terminal screen tail. It reaches one of five verdicts: done / still progressing / execution failed / insufficient evidence / unrelated to this task. Insufficient evidence is not immediately a failure; only repeated unclear rounds converge to "perception unclear", and you can confirm completion manually
- **Unclear goals get clarified first — no vague tasks**: kaya acts as the mission commander for your agents — when a request is unclear (missing goal or acceptance criteria), it questions and confirms with you in chat before writing anything into the ToDo List, so agents never chase a misaligned goal
- **Failures get attributed, not blindly retried**: when execution fails, kaya reads the screen and attributes the cause (unclear task / dispatch issue / agent stuck / environment); the failed entry shows the cause plus a suggested fix. In Auto mode kaya retries once with that attribution (except "unclear task"), then stops and waits for you; when the cause is an unclear task, it tells you exactly what information is missing and rebuilds a clarified task from your answer
- **kaya's coding methodology**: the commander follows its own working methodology (small steps / changes ship with tests / evidence-based acceptance / attribute before retry) when creating and verifying tasks; you can override it by writing your own into `~/.kaya/methodology.md` (its content is sent to the model along with the conversation — don't put secrets in it)
- Dispatch stays on the existing authorization model: the coding-mode master switch **and** the per-tab grant card are both required — but authorization no longer means extra back-and-forth: if the tab does not yet have a terminal-write grant when you click **Dispatch**, kaya also opens the grant card, and clicking **Allow** dispatches the task automatically. ToDo List contents stay on this machine and survive restarts; a task interrupted mid-flight safely returns to pending on the next launch

---

## Terminal and Tabs

**Basic Terminal Operations**

The right panel is your real shell: a login zsh on macOS (it loads your `.zshrc`, so aliases / environment variables / conda all match a native terminal); PowerShell on Windows (pwsh 7 preferred, falling back to the built-in Windows PowerShell 5.1; a git-bash user's `$SHELL` setting is respected and follows the original POSIX path). Common actions:

- Type a command and press **Enter** to run it
- Select text with the mouse, press **⌘C** to copy, **⌘V** to paste
- Scroll up with the scrollbar or trackpad to view past output

Commands you type in the terminal are **entirely yours**; the AI does not intercept or review them — that is your responsibility.

**Multiple Tabs**

kaya supports opening multiple terminals at once; each tab is an independent shell session:

| Action | Shortcut |
|---|---|
| New tab | **⌘T** |
| Close the current pane / tab (when focus is in the terminal) | **⌘W** or the × on the tab |
| Switch to the Nth tab | **⌘1** – **⌘9** |
| Previous tab | **⌘⇧[** |
| Next tab | **⌘⇧]** |
| Reorder tabs (when focus is on the tab bar) | **⌥⌘←** / **⌥⌘→** |

Each tab has its own working directory, command history, and scrollback. When a tab is closed, its command history is automatically merged back into the global history (on macOS it merges into `~/.zsh_history`; on Windows each tab's history lives in PowerShell's PSReadLine history and is merged into kaya's own global history file on close, leaving your shell history untouched).

**Font Size**

- **⌘=** or **⌘+**: enlarge by 1px
- **⌘-**: shrink by 1px
- **⌘0**: reset to default size

**Closing Confirmation**

Closing a tab (**⌘W** or × button) and closing the entire kaya app (red × in the corner) both show a confirmation dialog, to prevent accidental data loss.

**⌘Q Does Not Show Confirmation**

When quitting with **⌘Q**, macOS does not show a confirmation dialog, but kaya first tries to merge the local shell history of all tabs into `~/.zsh_history` and then exits. Where possible, prefer the red × or the menu Quit command over ⌘Q.

**Disconnected Tabs**

If a tab disconnects due to a network or process error:

- The tab status appears gray
- Open **⚙ Settings** → the **Shell** section of the **General** tab, then click **Reconnect** to re-establish the PTY for that session

**Tab Waiting-for-Confirmation Reminders**

When an AI CLI running in a tab (kimi code, claude code, etc.) pops up a confirmation that needs your decision (such as a command approval), kaya proactively notices and funnels the decision into the chat input — no need to watch the terminal yourself:

- A notification card appears in the chat window, showing which tab's which AI is waiting, with a summary of the confirmation; the card only offers **Dismiss** (if the confirmation content changes, it reminds you again)
- A reminder badge shows on the corresponding tab
- If the latest pending confirmation carries an option menu, a decision bar stays pinned above the chat input: it shows the decision prompt, what each numbered option means, and a "type the number to choose" hint
- You simply type the menu number in the chat input and send; kaya writes the corresponding keypress back to the AI CLI in that tab. Before writing, it re-verifies several things: the tab is still alive, the foreground process is still a supported AI CLI, the menu is still on screen, and the number is a valid option — only then is the single-digit menu number written through
- An out-of-range number, input that isn't a bare number, or having no pending confirmation at all is never intercepted — it is sent as a normal message
- Once you've handled it in the terminal, the card automatically flips to **Handled**

kaya never answers an AI CLI's own confirmation on your behalf. That is the CLI's own safety gate, and the decision is always made by you.

**Coding Mode (Proactively Watching Your Coding Agents)**

Turn on the coding-mode toggle in the top bar and kaya proactively watches the coding agents running in your tabs (Claude Code / Kimi Code / OpenCode / Codex, etc.) — no need to watch the terminal yourself:

- When an agent **pops up a confirmation and waits for your decision**, kaya automatically starts a chat turn, relays what decision is needed, and asks for your instructions
- When an agent **finishes its task (the screen goes idle)**, kaya proactively reads the result, summarizes the progress, and asks about next steps
- The watch is **read-only**: kaya only reports and asks — it never clicks a confirmation for you; the decision is always yours
- This toggle is also the **master gate for "driving"**: only while it is on can the chat side send instructions to a coding agent's tab (each tab still needs its own grant-card approval); turn it off and that capability is withdrawn **from the next turn onward** (the tool is no longer registered; a turn already in progress finishes with the state it started with). What remains active with the toggle off is just the passive notification cards and tab badges
- The confirmation summary carried into an auto turn is secret-scrubbed; each tab has a 60-second cooldown, and kaya never interrupts an ongoing chat turn (it follows up after the turn finishes)
- **Smart dispatch (v2.4.0)**: before relaying or dispatching an instruction, kaya evaluates the target agent's live state (busy / idle / waiting for you / what it's doing); instructions are always written through, but the result carries a status note — a "will queue behind current work" hint when the target is busy, and a gentle you-may-not-need-to-resend warning when the identical instruction repeats within 60 seconds; after writing, kaya also re-reads the screen to confirm the instruction was actually submitted, auto-resending Enter once if it's stuck on the input line
- kaya knows where every running agent lives (working directory, project label) and what it has been doing recently; ask "how far has it gotten" and kaya deep-reads that agent's terminal history on demand before answering — through the same secret-scrubbing pipeline
- **Fast decision layer (v2.3.8, on by default)**: proactive watching first passes a millisecond semantic gate — events clearly not worth interrupting you (like a repeated idle right after a report) no longer start a turn, and transient "still running" states are no longer announced either. Simple questions (chit-chat, one-liner Q&A) take a lightweight turn — faster responses, less quota. Decision requests go to exactly the model you picked for chat and pass the same scrubbing pipeline before sending; toggle it off under **⚙ Settings → Models**, effective immediately
- Note: coding-mode auto turns are sent to your selected model over the same path as your manual questions; with the toggle off, nothing is triggered at all

**Running-Agents Sidebar (Coding Mode Only)**

With coding mode on, a collapsible sidebar sits at the left edge of the chat panel (click the arrow handle on the edge to expand/collapse; the open state is remembered; drag the right-edge handle to resize the expanded panel; on narrow windows it becomes a drawer overlaying the chat instead of squeezing it; in terminal-fullscreen it stays docked at the window's left edge, still collapsible and resizable):

- See every running coding agent at a glance: each card shows the instance number (e.g. `kimi-code #2`, a global sequence shared across agent types), its current state (waiting for you / idle / running), and its tab and working directory; a running card also shows a summary of what the agent is currently doing
- Each card also carries a **status line**: current model, thinking level, permission mode, and context usage (kimi code shows all four; claude code can show the model, thinking level, and context usage plus the working directory, worktree, and branch; codex shows the model only; opencode shows the model, mode (Build / Plan) and thinking level — no context usage). Permission modes render in plain words (e.g. "Always ask / Ask when needed / Never ask"); when space runs out only the key segments survive — hover for the full text
- Each pane can carry an editable **project label** (click the project line on the card to edit it; persisted per pane — two panes in the same tab can carry different labels): when you run multiple projects in parallel it tells you which project each agent is working on, and it shows up on the card and in the `@` list
- One click on a card does three things: switches to that tab, binds the agent as the forward target of the current chat, and focuses the input box — then just talk to it in chat, the same channel as `/agents` binding; the click also asks the AI to read the screen and report the agent's latest progress (repeat clicks within 10 seconds don't re-report; while a chat turn is busy you only get a hint, no interruption)
- Cards in the "waiting for you" state list the agent's option buttons inline — answer with one click, no tab switch needed
- When collapsed, the handle shows a count badge; it turns into a warning color while any agent is waiting for a decision

**Per-instance runtime configuration and the Coding Agent workspace**

A running local agent can be configured for the **current instance**. Saving first queues the change; once that instance is idle and its identity has been revalidated, kaya applies it through the agent's native controls without restarting the agent or changing global defaults. Model, thinking effort, and Permission are applied and verified as separate fields. Nothing is written when the instance has changed, is busy or waiting for a decision, or cannot provide evidence that the change took effect. Remote agents remain read-only.

Availability is limited to fields with verification evidence: **Claude Code** supports model / thinking effort / Permission; **Kimi Code** supports model / thinking effort, but not Permission; **OpenCode** supports thinking effort / Permission, but not model; **Codex** currently exposes none of these runtime fields. Without a model-gateway connection, only the model field is disabled; other supported fields remain available.

The Coding + chat-focused view is a three-column workspace: running agents on the left, Chat as the main interaction surface in the center, and only pending decisions and grants on the right. Busy and status-bar changes update local cards. Explicitly enabling Coding reports the overall workspace status; idle and waiting-for-decision events are combined into occasional read-only follow-ups, deferred while Chat is busy. Turning the mode off, switching conversations, or pressing Stop cancels pending reports. Restoring the saved mode on startup does not repeat the overview. Chat stays mounted when switching views or turning Coding mode off, preserving composer drafts and pending images.

**Agents Panel**

Open **Agents** from the app menu (or click the Agents icon on the right side of the top bar) — it opens Settings and lands directly on the **Coding Agents** tab, the management center for coding agents. Installing, updating, and model configuration can also be done by asking kaya in chat (e.g. "install Claude Code for me") — both paths run the same engine:

- **Background install / update** of coding agents: kaya tries each available method in turn (npm > brew > the official script), with live progress shown on the card; a failed method automatically falls through to the next one, and only when **all** methods fail does it report an error — listing each method's own failure reason; "Install in terminal" remains as a manual fallback. Only one background install / update / uninstall task runs at a time — while one is running, the matching buttons on other cards are temporarily disabled; agents that cannot be updated (not installed via npm/brew) have the **Update** button disabled with an explanation
  - If the npm method hits a network-class failure, kaya first retries **once** via the npmmirror registry before counting it as failed and falling through
  - Each method's single attempt has a **10-minute overall timeout**: a timeout counts as that method's failure and falls through to the next method — only when **all** methods fail does the install report an error; if nothing seems to move, check the output tail (last 50 lines) on the panel card to tell slow from stuck
  - Uninstall is supported for npm/brew-managed installs (with an ownership check — an install not managed by that package manager is never touched) and removes only the program itself, keeping config and history; script-installed agents must be removed manually per the official docs. If an uninstall fails, "Uninstall in terminal" re-runs the exact command kaya attempted as a manual fallback
- **Custom command names (renamed agents are still recognized)**: if you renamed an agent's CLI command (e.g. you use `myclaude` instead of `claude`), add those names in the "Command names" field on the agent's card (comma-separated, full-width commas accepted). Once saved, install detection, foreground-process recognition, and the launch command all accept both the original and your custom names. Saving works even when nothing is currently detected — the status turns positive automatically once the command is on PATH. Process aliases of the agent's official binary are also recognized out of the box (e.g. after restarting a session with `kimi -r`, the kimi code process shows up as `kimi-code`) — no manual entry needed
- **Model connection (the "Model Gateway" card)**: the agent detail page's **Model Gateway** card handles model connection — an **Attach via gateway** toggle plus a **default model** picker (options = library models available for that agent's protocol family, grouped by provider; unselectable ones carry the reason). Adding, editing, and deleting providers and models all happen in **Settings → Models** — models are maintained in that one place. **This is the same library as "My models" on the chat side** — one provider carries three optional protocol endpoints (Chat / Anthropic / OpenAI) plus a model list, each agent picks what it needs, and the same provider is visible and usable for chat in Settings. Once connected, the agent's config converges to the gateway form: the endpoint always points at the local loopback gateway, the model shows a friendly name (e.g. "Deepseek (deepseek-flash)"), and **neither the real endpoint address nor the API key ever lands in the agent's config file**; turning the toggle off strips kaya's configuration and restores original managed keys from the pre-takeover snapshot; original provider/model entries are filled only when missing, with current entries of the same name taking precedence
  - **Protocol-based attachment**: Claude Code uses Anthropic endpoints; Kimi Code / OpenCode use Chat-compatible endpoints: an enabled, nonempty OpenAI endpoint declaring chat/both takes priority, falling back to the Chat endpoint only when that slice is unavailable. Binding a default Codex model in Settings requires an enabled OpenAI endpoint declaring Responses or both. Once attached, Codex’s in-agent picker can use Responses-to-Chat adapted models from Chat-only providers. The agent catalog and attachment diagnostics show availability and explain incompatible models
  - **API keys live in the system keychain (macOS Keychain / Windows Credential Manager)** (shared with the chat-side models; the library file itself holds **no** keys). When kaya edits a config from chat, a grant card asks for your confirmation before anything hits disk; secret values are never echoed back into the chat; deleting a provider **also deletes its key** (the confirmation dialog says so)
  - **Codex isolation and optional global takeover**: by default, the separate kaya profile affects only Codex launched from kaya or explicitly selecting that profile. Enabling **Global takeover** also points the official configuration at the gateway, affecting plain Codex and sessions launched by the ChatGPT desktop app; confirmation is required. Disabling it, stopping the gateway, quitting, and crash reconciliation share the same snapshot restore process. Failed restoration keeps the snapshot and reports the problem
  - **Model pickers** list library models available for the agent protocol, grouped by provider with names, context windows, and capability information. **Show bound model only** narrows the list. Library additions, removals, and capability changes propagate to the agent; kaya manages this model-selection surface while attached
  - **The model library is the single source of truth and the only writer, and one provider can serve several agents at once**: which agent is using which provider's which model is recorded explicitly (not inferred by comparing config values), so editing or deleting a provider acts directly on the agents using it. kaya's AI assistant cannot change the model-identity keys in an agent's config file either (model name / endpoint address / key) — such edits are refused and it is pointed at the model library instead
    - After you edit a provider (endpoints / model list / API key) the config of every connected agent **follows automatically** — no second action needed. Clearing the API key also invalidates the corresponding key in the gateway
    - After you delete a provider, the agents connected to it drop kaya's configuration and fall back to their own defaults — they stop using the deleted model and key
    - **Exception**: when **kaya did not claim the config** (it carries no kaya-written marker — e.g. it was never connected; or the provider was deleted before claiming could happen), the kaya settings left behind in that agent's config file are **not cleaned up automatically** — connect that agent once to converge it
    - Unattached agents are not touched. Once explicitly attached, kaya backs up and removes or hides non-kaya model/provider choices that already existed at takeover; entries added afterward are retained. Detaching merges the original entries back, preserving current entries with the same names. Other fields follow each agent’s managed-key boundaries
    - **In use** only counts agents truly connected via the gateway (a placeholder with the toggle off does not count); if an agent's config file was edited by another tool, the detail page shows a "config edited outside kaya" badge — click **Re-attach** to re-apply
    - A coincidentally matching endpoint does not mark a hand-written config as in use; bindings and markers determine management. Explicit attachment captures original values first. User endpoints or model values that cannot be shown recoverable are left unchanged for that sync, with a diagnostic. A corrupt original snapshot is retained rather than replaced by a new capture
    - If one agent's config file is broken, only that agent is affected: the others are still updated, and the broken one is retried on your next action
    - **Boundary**: removal restores from the per-key snapshot taken before kaya first took over; configs taken over by older versions (before snapshots existed) have no snapshot, so removal degrades to stripping only the keys kaya wrote — restore the old values by hand if needed

**Config optimization**

Choose **Optimize** in the **Config optimization** card on an agent's detail page. kaya reads the scrubbed current config once (current values of writable keys, bound-model library metadata, deterministic first-pass issues) and asks the model for sourced suggestions. Analysis is **read-only**; AI suggestions and kaya's built-in recommendations merge into **one checklist**:

- **AI suggestions** each carry a source (a real file in the official docs snapshot / a model metadata field / a built-in tuning key) and pass a fail-closed Rust validator: unknown agent, refused keys, keys outside the allowlist, non-"factual fix" changes to values you set explicitly, misreported grades, fabricated doc paths, out-of-range values, and no-op changes are all filtered and shown transparently as three groups — selectable / manual / blocked by the safety gate.
- **Built-in recommendations** are kaya's own deterministic tuning (status line / event hooks / auto-compact, etc.), only appending kaya-owned entries, selected by group.

Tick items and choose **Apply selected**; the two paths run independently and **never swallow each other's errors**:

- AI items: one-time authorization (bound to the proposal hash + the selected-subset hash, burned after use, cleared on restart) → deep-merge write (with a `.bak` backup) → **per-key read-back verification**; any mismatch rolls back from the backup automatically, and the card reports ✓ n/m plus failure details.
- Built-in items: applied as a whole via `agents_tune_apply`; the group header always shows **Revert**, greyed out with an explanation when there is nothing to revert.
- Each path toasts and leaves its own "not applied" trace; one path failing never cancels the other.

**Semantic verification**: reading the file back only proves "it was written", not "it took effect". Keys that map to a status-line dimension enter "pending confirmation"; once the agent restarts and the value is parsed, they flip to "effective". Timing out flips to "unconfirmed" — **no automatic rollback**; that call is yours.

**Undo** checks the current file hash against what this application wrote; if you edited it afterward, kaya preserves your edits and refuses to overwrite them. AI undo state lasts only for the current app process; built-in recommendations keep their separate **Revert**. Apply and rollback on the same config file are serialized per path, so they cannot overwrite each other. Script-based recommendations are unsupported on Windows; Kimi Code uses its official default status line.

Model names, endpoints, credentials, and thinking default efforts remain managed exclusively in **Settings → Models** — they cannot be changed through config optimization. Codex's `model_reasoning_effort` and claude's supported capabilities / effort level are on the refused-key list because the library projects them.

- Over IM channels (Feishu / Slack / WeCom): only config read/edit is available (grant cards included); install, update, and the management center's status queries are not available there (a running agent's live status can be queried read-only)

---

## Let the AI Read a File

By default the AI cannot read files on your computer. If you want the AI to help you analyze some code, view a log, or explain a configuration file, you need to first place the file in the trusted directory.

**Steps**

1. Create or use the `~/.kaya/` directory in the terminal:

```bash
mkdir -p ~/.kaya/my-project
cp ~/my-project/main.ts ~/.kaya/my-project/
```

2. Tell the AI the file path in the chat:

```
Read ~/.kaya/my-project/main.ts and explain what it does
```

3. The AI calls an allowed read command (such as `cat` or `head`) to read the file and shows it to you

4. Delete the copy when you no longer need it:

```bash
rm -rf ~/.kaya/my-project
```

**Allowed Read Commands**

Under `~/.kaya/`, the following commands are available:

- `cat` / `bat`: read file content
- `head` / `tail`: read the beginning or end
- `grep` / `egrep` / `fgrep` / `rg` / `ag` / `ack` / `sed -n`: search file content
- `sort` / `uniq` / `tr` / `cut` / `nl`: rearrange and transform text
- `strings` / `xxd` / `hexdump` / `od` / `base64` / `base32`: view binary or encoded content

On Windows, the PowerShell equivalents `Select-String` (`sls`) and `findstr` are available too, with the same rule that every path must live under `~/.kaya/`.

Even commands that return only counts or hashes, such as `wc`, remain subject to trusted-read checks when given file paths. They cannot be used to inspect file content or sizes outside `~/.kaya/`.

**Disallowed Operations**

Even under `~/.kaya/`, the following remain blocked:

- Modifying or moving existing files (`sed -i`, `mv`, etc.)
- Deleting files
- Executing file contents

The one write exception: redirecting command output into a file under `~/.kaya/` (e.g. `echo hello > ~/.kaya/note.txt`) is allowed, but the target must carry a safe file extension — extension-less targets and dangerous suffixes (`.sh`, `.exe`, etc.) are still blocked. Beyond that, the AI can **read, not modify** existing files. This is intentional.

**Downloading Files**

The AI can also download files from the network into `~/.kaya/` (for example `curl -o ~/.kaya/...`). Dangerous file types (such as `.sh`, `.exe`, `.dmg`) are blocked.

**Reminders**

- Do not put keys into `~/.kaya/` just because there is scrubbing, hoping "the AI will look at them"
- Delete files under `~/.kaya/` promptly when no longer needed
- Do not place symbolic links to sensitive external files in the trusted root. Some path checks are lexical and do not guarantee resolution of every symlink; see the known boundaries in privacy design §5.1

---

## Web Search

kaya can let the AI search the web through Tavily. Tavily is optional and disabled by default.

**Enable**

1. Sign up at [tavily.com](https://tavily.com) and obtain an API key
2. In kaya, open **⚙ Settings** → the **Web Search** section of the **General** tab
3. Paste the Tavily API key
4. Save

**Usage**

Once enabled, ask the AI to search in the chat:

```
Search for the latest features in React 19.
Look up the latest DeepSeek model news.
Search and summarize the main points of this English article.
```

The AI calls the search tool, retrieves results, and summarizes them for you.

**Privacy Note**

Tavily is an **optional additional outbound path for web search**: once enabled, it sends your scrubbed question text to Tavily. What Tavily receives is only the text of your question to the AI — not your files, not your API key. **Disable Tavily to close this outbound path**.

**Trustworthiness Reminder**

Search results are **untrusted input from the internet** — they may be outdated, incorrect, or biased. The AI's summary still needs you to verify key facts. Enabling Tavily does not bypass kaya's content protection: page content goes through scrubbing before entering the model context.

---

## Use kaya via Feishu / Slack / WeCom

Since v2.1.0, kaya can bridge IM messages to this machine: message the bot in Feishu, Slack, or WeCom (Enterprise WeChat) and your local kaya LLM handles it and **streams a reply** — no need to have the kaya window open.

**How to enable**

1. Open **⚙ Settings** → the **IM Channels** tab (also reachable directly from the system menu: kaya → **IM Channels…**) and pick a platform — **Feishu** and **Slack** are open for setup in this version; **the WeCom entry is currently greyed out as "(coming soon)"** (existing WeCom configs are preserved and will be re-enabled in a later version)
2. Feishu: **recommended — click "Connect via Feishu QR"** (Feishu only): scan the QR code with the Feishu mobile app and confirm — kaya automatically creates the app and fills in the App ID plus the creator's open_id in the allowlist; the App Secret goes straight into the **system keychain (macOS Keychain / Windows Credential Manager)**, never to disk, never echoed back
   - Manual configuration (fallback): create an app yourself on the [Feishu Open Platform](https://open.feishu.cn) or the WeCom admin console, then enter the Feishu App ID + App Secret, or the WeCom BotID + Secret
3. Slack (**manual only, no QR code**): create an app at [api.slack.com/apps](https://api.slack.com/apps) → enable **Socket Mode** and generate an App-Level Token (`xapp-` prefix, scope `connections:write`) → add the OAuth scope `chat:write` → subscribe to the `message.channels` / `message.groups` / `message.im` / `message.mpim` events under Event Subscriptions → install the app to your workspace to get the Bot Token (`xoxb-` prefix) → back in Settings → IM Channels, fill in both tokens (both go straight into the system keychain) plus a member ID allowlist
4. Configure the **admission allowlist**: one user ID per line (Feishu `open_id`, starting with `ou_`; Slack member ID, starting with `U` — copyable from the member's profile page; WeCom member userid); Feishu QR provisioning locks it to the creator automatically
5. Click **Start channel**; once the status shows "running", message the bot from your IM. **If a permission error appears**, grant the app the required permissions on the platform console (Feishu: enable the im:message scopes and publish a version; Slack: double-check the scopes and event subscriptions)

**Key points**

- **Direct long connection to the platform**: Feishu official WebSocket, WeCom official openws channel, Slack Socket Mode WSS — **no public IP or callback server needed**
- **Allowlist denies strangers by default**: messages from users not on the list are dropped before ever reaching the LLM; an empty list prevents startup
- **Same privacy pipeline**: IM messages go through the same five-gate pipeline as desktop chat; IM turns use essentially the same tool table as the desktop (terminal read / task list / memory retrieval included), with three deliberate exceptions — schedule management, the cross-agent task queue, and AI-side task creation are not offered over IM. You can say "remember: …" directly in IM (the memory-write tool is registered only for remember-intent turns; writes are adjudicated automatically by evidence and take effect immediately, with the real outcome reported — an archived write says plainly that it is not active), and memory-retrieval output is scrubbed too. `terminal_write` is still governed by the coding-mode master switch — that switch is an atomic flag shared between desktop and channel; toggling it on the desktop is pushed to the channel immediately via `channel_set_coding_mode`, taking effect from the next IM turn
- **Desktop → IM mirroring (opt-in, off by default)**: once enabled, the desktop chat stream is pushed in real time to your chosen IM chat, and your own inputs are echoed as "你:…"-prefixed text; IM replies also appear in the desktop chat panel. Starting a new desktop session migrates the mirror automatically and pushes a "new session started" notice. The toggle lives in two places — the "Mirror desktop chat to Feishu" checkbox in Settings and the mirror icon next to coding in the top bar; both are the same switch and take effect immediately
- **Feishu / Slack authorization cards**: when an IM turn needs terminal_write / agent-config approval, Feishu / Slack shows an interactive card with Approve / Deny buttons (Block Kit interactive buttons on Slack); decisions share the same grant store as desktop approval cards (mutually recognized); the operator must be on the admission allowlist. Once approved, kaya **automatically runs a follow-up turn** asking the AI to retry the paused operation — no need to wake it up with another message
- **Proactive agent-status notifications**: with mirroring on, the coding agent's waiting-for-input / idle-done / online / exited transitions are pushed to the mirrored chat (Chinese or English per the channel locale, with per-event cooldown to avoid flooding). kaya's automatic turns (timing questions / status reports, etc.) appear on IM only as one-line summaries rather than the full instruction text, and a proactive notification plus its turn mirror for the same event is sent only once
- Session mappings and conversation context are stored under `~/.kaya/`; **IM incoming / outgoing rows are not yet wired into audit JSONL**. The LLM uses the model source picked in the top-bar model switcher
- **Honest caveat (WeCom)**: WeCom mirror echoes and proactive notifications depend on an active callback registration and may be silently dropped in pure-desktop scenarios (no IM turn in flight); only Feishu was verified on a real device in this round — Slack has not yet been verified against a real workspace

See the "IM channel entry" and "IM channels & mirroring" sections of [docs/PRIVACY.en.md](./PRIVACY.en.md) for details.

---

## Remote Access (Cloud Relay)

Since v2.3.2, kaya can be viewed and operated remotely **from a browser**: conversations, running-agent status, and approval prompts stay in real-time sync with the desktop — keep work moving even when you're away from your computer.

**How to enable**

1. Open **⚙ Settings** → the **Remote Access** tab
2. Make sure you're signed in to your yunhou account with an active subscription (the toggle is disabled with guidance otherwise)
3. Turn on "Enable remote access"; once the status shows "running" and the connection state is "online", you're set
4. In a browser, sign in to the yunhou website and open the "Remote Control" page to reach this device

**Key points**

- **Opt-in, off by default**: no remote path exists until you enable it; turning it off disconnects immediately
- **In-memory relay only**: conversation content transits yunhou servers, which forward it in memory without persisting anything; the remote end sees exactly what the desktop shows. **API keys never transit the relay** — this machine still connects to providers directly; everything is TLS-encrypted
- **Same privacy pipeline**: remote turns go through the same five-gate pipeline as desktop chat (input scrub → command whitelist → safety check → isolated execution → output scrub); write operations such as `terminal_write` share the same grant store, so approvals on the remote end and on the desktop recognize each other
- **Device identity**: a device ID is generated on first enable (copyable); the device name is customizable (≤64 characters) to tell multiple machines apart
- **Status at a glance**: the panel shows running state, connection state (connecting / online / offline with auto-reconnect / auth failed), the last error, and last activity times; if the paid entitlement lapses, remote access stops automatically and says so
- Device identity is stored in `~/.kaya/remote/device.json` (0o600 permissions); session mappings and conversation context stay local, while **remote incoming / outgoing rows are not yet wired into audit JSONL**

See the "Remote Access (Cloud Relay)" section of [docs/PRIVACY.en.md](./PRIVACY.en.md) for details.

---

## Cross-Device Coordination

Two kaya desktops under the same account can **hand work to each other**: dispatch a task from machine A's chat / ToDoList to a coding agent running on machine B, and B's agent status, evidence, and grant requests stream back in real time. Running instances on B appear in A's sidebar and `@` menu with a **device-name badge**. Clicking a remote card selects that agent and binds Chat while keeping the local terminal tab unchanged.

**How to enable**

1. Sign both machines in to the same yunhou account (cross-device coordination reuses the remote-access relay connection; each machine needs an active subscription)
2. On the **executor** (the machine receiving tasks), open **⚙ Settings** → **Cross-Device**, turn on the master switch, and add the initiating device to the **allowlist** (optionally scoping it to specific agents)
3. On the initiator, pick the remote target (device + agent) on a ToDoList task and dispatch; the dispatch text goes through the executor's grant card

**Key points**

- **Executor admission is fail-closed and off by default**: master switch off out of the box plus a per-device allowlist; execution-directed requests from non-allowlisted devices are rejected. The allowlist lives in `~/.kaya/remote/peers.json` (0o600; a corrupt file resets to off). **Discovery and status frames do not pass through this execution-admission gate**: once remote access is enabled, same-account devices can exchange discovery and agent-status summaries. Disabling coordination or omitting an execution allowlist does not make the device invisible
- **Both ends can approve; authority sits on the executor**: writing into the remote terminal requires a grant. The grant card shows on both ends (labeled "from device" / "target device"), either end approving takes effect, and you can remember "never ask again for this device". Dispatch text is submitted as agent input, not raw command injection — control characters are rejected
- **Read output on demand**: when the executor allows Conversation relay and both versions support it, read the remote screen or report without calling the peer's model. Admission scope and instance identity are checked before and after the read; content is scrubbed and capped at 8 KiB. Changing instances clears the previous output
- **Answer the current menu**: supported remote decision menus accept choices 1–9 from the sidebar or workspace, and appear in Chat as pending decisions that you can answer by typing a digit. Each answer is bound to the current instance and a single-use menu ticket; a local answer revokes the old ticket, and expired or consumed tickets and replaced instances cannot receive the answer. A timeout reports an unconfirmed result without automatic retry. Exit or disconnect closes pending notices. Older peers show an upgrade prompt; this entry point does not provide arbitrary keystrokes or runtime configuration changes
- **Remote state snapshots and page recovery**: reloading restores each device's connection baseline, so agents that appear later are still picked up even when that device initially has none. A brief disconnect resumes through the current handshake, while retired connections and late events cannot reopen old state. Recovery snapshots and live events join at the same watermark, preventing an old snapshot from overwriting an exit, disconnect, or instance replacement. Display projections are bounded and kept only in local memory
- **The relay stays a dumb pipe**: device-to-device frames transit the same relay connection without parsing or persisting payloads; frame content passes through output scrubbing before leaving the machine
- **Disconnect semantics**: an offline peer marks the item "reconnecting"; on reconnect kaya reconciles automatically (still running / exited / grant pending). A confirmed still-running peer is never re-dispatched; a lost task record is reported as such, and you can retry or mark it done manually
- **Server dependency**: cross-device coordination requires the yunhou relay server's device routing and sender-identity stamping; until the server is upgraded the two devices cannot see each other and the feature stays inert (enabling it on the client creates no device-to-device path)

**Ask or forward from chat**

The receiver must separately allow **Conversation relay** for the initiating device; it is off by default. @ a remote agent in chat to send the request to the peer's kaya, which can read scoped agent state, reports, and terminal tails or forward an approved instruction. A missing forwarding grant opens the existing approval card on both ends and reports that authorization is pending. Admission and the bound instance are checked again at approval and at the actual write. After an agent restarts, old requests cannot be forwarded to the replacement; bind it again. This mode does not expose shell, file-editing, memory-writing, or web-search tools. Read access is rechecked against current running agents when tools execute, and no machine-wide agent/task situation is injected. Approved global rules and core memory continue to follow the receiving device's local conversation-context contract.

Both ends show the conversation with its device of origin. The initiator receives the final reply rather than a live replay of the peer's reasoning and tool output. Offline, refused, and timed-out requests reach a terminal state. ToDoList dispatch retains its separate task/approval flow.

See the "Cross-Device Coordination" section of [docs/PRIVACY.en.md](./PRIVACY.en.md) for details.

---

## Thinking Mode

**What Thinking Mode Is**

When enabled, before the AI gives its final answer it performs a **chain-of-thought reasoning**: it shows you its reasoning in a collapsible paragraph, and then answers your question.

**Why Use It**

- **Explainable**: you can see why the AI reached this conclusion
- **Auditable**: reasoning is recorded separately and can be replayed later
- **More accurate**: for complex problems, answers with reasoning tend to be more accurate

**Cost**

- **Slower**: the AI needs time to think before answering
- **More expensive**: more tokens consumed, higher API cost
- **Potentially longer**: reasoning content takes up space

**How to Enable**

Thinking capability is declared **per provider / per model** now; there is no single global switch:

1. **Chat-endpoint thinking toggle** — Settings → Models → provider detail, tick **Thinking mode** next to the chat endpoint. This decides whether that endpoint forwards reasoning parameters; kaya adapts to the protocol and current model ID (DeepSeek injects `reasoning_effort` + `thinking`, OpenAI o-series forwards `reasoning_effort`, OpenRouter injects `reasoning: { effort: "high" }`, GLM / Qwen inject their own switches, and DeepSeek reasoning models on Ollama / Custom endpoints also inject `reasoning_effort` + `thinking`; other models get no injected parameters).
2. **Per-model thinking declaration** — if the catalog data is wrong, or you want to turn off a model the catalog says supports thinking, tick / untick **Thinking** on the model row. This is a **tri-state**: auto (catalog fallback) / supported / not supported, and **an explicit declaration wins over the catalog**. **Clear declaration** removes the thinking and effort declarations together and returns to automatic inference.
3. **Thinking efforts** — when a model supports thinking, the row shows effort chips (**minimal / low / medium / high / xhigh / max**) plus a **default effort** dropdown. The default is "decide from catalog"; clicking a chip freezes the current effective set as an explicit declaration, and clearing returns to automatic. These efforts project into each agent's native keys, so an agent's status line `thinking: <level>` matches actual behavior.

**Two different things both called "thinking"**: whether a chat request carries reasoning parameters is decided by the **chat-endpoint toggle plus the current model ID**. The model row's thinking tri-state / efforts serve coding-agent config projection, catalog badges, and config-optimization analysis; they do not become the thinking boolean in a chat request. After changing a declaration, a running agent sees the new value only on the next sync (next save / re-attach / agent restart).

**Switching the Reasoning Model**

The provider detail page no longer has a **Reasoning Model** editing field. Existing `reasoning_model` data is consumed only by the **AI judge** in audit evaluations (to pick the judge model); chat requests do not use it or use it to switch the actual model ID.

To switch which model chat uses, change the model selected under **Settings → Models**. Reasoning parameters are adapted from the endpoint protocol and current model ID automatically, so you do not need to switch to a separate "reasoning-only model".

---

## Sessions and History

**Session Persistence**

kaya automatically saves your conversations. Three layers stay in sync:

- UI messages (the chat panel) are saved in the app's local state
- LLM context (for the AI to review) is saved in `~/.kaya/conversations/`
- Audit logs (the detailed event stream) are saved in `~/.kaya/audit/`

After refreshing the app or restarting your computer, conversations and settings are restored automatically.

**New Session**

Click the **↻ New Session** button in the top right:

1. Stops any turn in progress
2. Clears the current UI messages
3. Starts a fresh session context
4. Begins a brand-new conversation

**View Past Sessions**

The **New Session** button is a plain icon button with no session list next to it. There are two ways to view history: use the **`/sessions`** command in the chat input to list past sessions and click a card to view them (**read-only** — you cannot resume the context and keep chatting); or open **⚙ Settings → Audit** for the full event stream.

**Compact the Context (/compact)**

When a conversation grows long, type **`/compact`** in the chat to compress the current conversation's context into a summary and continue from there (the chat history you see stays unchanged). Before compacting, kaya distills memory once — important facts from the full history land in the memory store first and are not lost to compaction. Progress shows in the message stream (distilling memory → generating the summary → done). The command requires sign-in; compacting with the builtin model requires a subscription.

**Clear Data**

kaya has no per-session delete UI. To clear all local data, run in the terminal:

```bash
rm -rf ~/.kaya/
```

This also removes conversation context, audit logs, the memory store, token usage stats, gateway config and privacy records (all of them live under that directory). **UI messages and settings are not under `~/.kaya/`** — they live in the app's local storage; to wipe those too, clear the app's local data (or start a new conversation to overwrite them).

**Tabs and Exit Behavior**

- **Close tab** (**⌘W**): confirmation dialog appears; on confirm, the tab closes and its shell history merges into `~/.zsh_history`
- **Close the entire kaya** (red ×): confirmation dialog appears; on confirm, all alive tabs' shell history merges into `~/.zsh_history`, then kaya exits
- **⌘Q**: no confirmation dialog; kaya first tries to merge local history into `~/.zsh_history`, then exits. Where possible, prefer the red × over ⌘Q

---

## Memory

kaya can **remember across conversations** — your preferences, important decisions, and project constraints survive into the next new chat.

- **What gets remembered, and how**: valuable facts from conversations are distilled into *memory candidates*, which the Agent **adjudicates automatically by evidence class** — facts may be stored from explicit instructions, statements and repeated patterns; a rule becomes always-present only when your own message carries an explicit remember-rule instruction and its stripped content matches the candidate exactly. A real message id or model-declared evidence alone is not sufficient proof. Content the assistant inferred on its own never enters active memory (at most it is archived to cold storage); when rule capacity is full the new rule is archived rather than evicting one of yours. **There is no review queue** — writes take effect immediately, and you can review, edit, roll back or permanently delete entries afterwards in Settings → Memory
- **Just say "remember"**: explicit phrasings like "remember: …", "keep in mind …", or "don't forget …" are all recognized. The instruction does not have to start the message — mid-sentence or sentence-final forms such as "run the tests first. remember: release Friday", "remember this point", or "release Friday, remember" are recognized too. When the entire message is the remember instruction, kaya stores it locally (no LLM quota spent, no login required); a mixed instruction like "remember X, then do Y" or the mid-sentence forms above is distilled by the AI during that conversation. Secrets in remembered content are redacted before storing
- **Three layers: core / semantic / episodic**: memories pinned to *core* travel with every conversation (up to 24 entries); the rest live as episodic memory that the AI retrieves on demand without costing context every turn; when episodic memories on the same theme keep recurring, consolidation automatically generalizes them into a *semantic memory* (e.g. several "agents.md needs trimming" entries become one long-term conclusion) — the generalization takes effect directly. Semantic memories never decay, and the original entries they were generalized from are kept, with the member list always viewable; little-used episodic memories naturally decay and get archived over time, and ones that get used again are reinforced
- **Rules vs facts, global vs project**: memories come in two kinds — **rules** are behavioral constraints that stay present every turn (e.g. "run tests before committing"); they take effect only when your own message carries an explicit remember-rule instruction and the detector's stripped content matches the candidate exactly, then travel with every conversation without retrieval (capped at 16 global, 8 per project); **facts** are background information retrieved on demand. Any memory can also be bound to a **project scope**: say "remember: project: …" or switch an entry to "this project" in Settings → Memory, and it only applies to sessions under that directory (project rules are injected with a "(this project)" marker). The AI also sees a compact **memory index** every turn (one-line summaries), so it knows what it can look up and fetches details only when needed — at minimal context cost
- **Inferred content never takes effect silently**: candidates the assistant inferred never enter active memory — they are kept as *archived* cold storage when they clear the long-term-value bar (or are rule-shaped), visible via "Show archived" in Settings → Memory and restorable with one click, and are **rejected and never stored** when below the bar, empty, or collapsed to a single secret placeholder. The automatic lifecycle only adds / reinforces / generalizes / archives / supersedes reversibly, and **never deletes physically on its own**
- **Recent automatic changes**: Settings → Memory has a collapsed "Recent automatic changes" section where you can look back at recent add / reinforce / supersede / generalize / archive records; the memory timeline is shown newest-first
- **When distillation happens**: automatically when a conversation ends; periodically at app launch and while the app keeps running (on by default, can be disabled in Settings → Memory); or manually via "Consolidate now" in Settings → Memory. Distilling with the builtin model requires a subscription; BYOK uses your own provider configuration; a local Ollama model means nothing leaves the machine. Memory starts from the moment the feature is enabled — earlier conversations are never mined retroactively
- **Deletion is forgetting**: all memories live only on this machine in `~/.kaya/memory/`. "Discard" is a logical delete (an auditable trail is kept); "Delete permanently" on an entry erases its content and all historical versions, unrecoverable
- **Retrieval summaries carry evidence and conflicts**: every summary the AI retrieves states its evidence class (explicit user / user-asserted / repeated pattern / assistant-inferred / semantic cluster), and unresolved conflicts carry the other entry's id — when a new statement conflicts with an old one but does not qualify for a legal supersede, **both are kept** and the AI judges in context; text similarity alone never overwrites an older memory

---

## Scheduled Tasks

Let kaya start chat turns on a timetable — for example, have it check on your agents' progress every morning, or summarize what changed in the terminal every so often.

- **Create and manage**: open **⚙ Settings → Schedules**, write what to do (the prompt) and pick a cadence — **interval** (every N minutes) or **daily** (local time HH:mm); an optional **run limit** caps how many times it fires (empty = unlimited; the task auto-disables when the limit is reached, and re-enabling restarts the count). Tasks can be toggled and deleted; editing is not supported — delete and recreate
- **Fires automatically**: when due, kaya starts a chat turn with that prompt; the process and result appear in the message stream and the audit log as usual; if a chat turn is busy it retries later instead of silently dropping the day
- **Same permission gates**: scheduled turns go through the same gates as manual questions — sign-in is required, and the builtin model requires an active subscription; when the gates are not met, that cycle is skipped silently without popups
- **Local only**: schedules stay on this machine and survive restarts; a daily task fires only once its time of day arrives (one created after today's time has passed starts tomorrow)

---

## Audit Records

kaya automatically records audit logs for **sources wired into auditing** (such as desktop chat and scheduled turns), so you can replay what the AI did in those sessions. IM / remote incoming and outgoing rows are not yet wired into audit JSONL.

**Open the Audit Page**

**⚙ Settings** → **Audit**.

**Sessions List**

The left side lists all sessions, sorted by time. Click any session to view its event stream.

**Event Types**

- Session start
- Each of your inputs (with scrub results)
- Each AI tool-call decision (why allowed or denied)
- Tool-call completion (with stdout, stderr, exit code)
- Each AI reasoning chunk
- The six config-optimization nodes: proposed / granted / applied / rolled back / undone / apply failed (only agent, proposal id, item count, and **key names** — never values)
- Gateway scrubbing events (`gateway-scrub`): category counts, byte counts, and the actual route only — **zero content**
- Error events

**Run Evaluation**

Click **Evaluate** on a session; kaya performs three passes:

1. **Rule replay**: re-runs current policy rules against all recorded tool-call decisions. Any deviation from current rules is flagged
2. **Anomaly heuristics**: checks 5 anomaly patterns — "file read without scrub", "network output without scrub", "environment-variable output without scrub", "tool call missing preceding reasoning chunk", "large Tavily output without scrub"
3. **AI judge**: an independent AI (always in thinking mode) gives the final review of the entire session

**Audit Logs vs App Logs vs Privacy Details**

- **Audit logs**: LLM-related events from sessions whose sources are wired into auditing, kept for replay. They **do contain scrubbed text** (your inputs, the first 2,000 characters of tool output, the model's answers and reasoning chunks) plus decision metadata; the only **content-free** audit event is the gateway scrub event `gateway-scrub`
- **App logs**: cross-session diagnostic logs for troubleshooting kaya itself (including trim / drift / gateway-upstream-timeout warnings)
- **Privacy details**: per-hit tokenization / warning / restoration records from the gateway — **the only local store that holds gateway-matched original text**, viewable under Settings → Gateway → Privacy statistics

The three are independent; do not confuse them.

---

## Settings Reference

Open **⚙ Settings**. On wide windows the left side shows grouped navigation — **Coding** (Models / Coding Agents / Gateway), **General** (General / Memory / Shortcuts / Schedules), **Connections** (IM channel / Remote access / Cross-Device), **System** (Account / Audit / Logs & Debug); narrow windows fall back to a top row of tabs. The **Models** tab opens by default. **Web Search, Shell, Language, Scrubbing** are all sections inside the **General** tab, not standalone tabs; the **Models** tab is the model library (see below); **Logs & Debug** is the former App Logs tab merged with the dev-only Debug section.

**IM Channels**

Bridge Feishu / Slack / WeCom messages to your local kaya: configure the platform credentials (stored in the system keychain: macOS Keychain / Windows Credential Manager) and the admission allowlist, start / stop the channel, and view its running status. See [Use kaya via Feishu / Slack / WeCom](#use-kaya-via-feishu--slack--wecom).

**Coding Agents**

Install / update / uninstall coding agents, set custom command names, and bind models in this tab (the former standalone Agents panel, now merged into Settings). See "Agents Panel" above.

**Models (Settings → Models tab)**

The full model-library description is in **Model Library and Providers** above. Here is just the layout:

- **Left column providers / right column detail**: **+ Add Provider** picks from the 200+ catalog or a custom one; the detail page manages authentication, the three protocol endpoints, the model list, and per-row metadata.
- **Decision layer** (global switches on the same tab): **fast decision layer** (simple questions take a slim turn; terminal events pass a semantic filter) and **command confirmation mode** (ask every time / auto-dispatch queued tasks). Both take effect immediately, need no save, and do not affect terminal-write grant cards.
- **Usage**: the tab header switches between **Library | Usage**.

**Usage statistics**

**Settings → Models → Usage** shows token / request totals for the last 90 days (default window), per-model and per-agent sums, with day / week / month buckets, agent filtering, and manual refresh. Chat and the gateway **share one accounting path**; records contain provider / model / agent identifiers and actual counts, never request bodies, and missing upstream usage is not invented.

- Auxiliary calls outside the agent loop (fast decision, task distillation, memory consolidation) are excluded; entry routing itself is not counted, while the slim turn it triggers is a main chat turn and is counted — so the numbers may be lower than a provider bill.
- Events stay locally for 180 days. **These statistics are not a provider bill or balance.**
- Upload scope, consent and disable behavior are described under Account below and in the [privacy design](PRIVACY.en.md).

| Item | Description |
|---|---|
| Provider | LLM provider: from the 200+ catalog, or a custom OpenAI-compatible endpoint; multiple accounts per provider allowed |
| Authentication | API key or OAuth sign-in; credentials live only in the system keychain, and the library file holds **no** keys |
| Endpoints | A provider may enable **Chat / Anthropic / OpenAI** endpoints at once; models belong to protocol families |
| Model list | Catalog pre-fill plus manual rows; online refresh; all models of a provider share one credential |
| Model metadata | Context window, max output, image input (auto / supported / not supported), thinking (auto / supported / not supported), thinking efforts and default effort, video / audio input |
| Existing `reasoning_model` data | Consumed only by the AI judge in audit evaluations; there is no detail-page editing field and chat requests do not use it (see Thinking Mode) |

> Note: **Max context messages** controls how much history the agent loop keeps (system prompt + the most recent N messages); the persistence cap is 200 messages per conversation.

**Gateway (Settings → Gateway)**

Mechanism and boundaries are in the **Model Gateway** chapter above; here is what the panel offers:

- **Enable toggle + status line**: running · port / stopped / held by another instance (dev and prod gateways are mutually exclusive; when held, writes are rejected and the holder is named).
- **Scrub level**: gateway-specific, strict by default (named secrets only; ordinary PII is left unchanged), balanced / aggressive available. User input is always aggressive; tool output defaults to aggressive and is adjustable under General.
- **Claude unmanaged by kaya**: when on, kaya writes no managed config for Claude Code (existing managed config is removed), so its traffic bypasses the gateway and is not scrubbed.
- **Reject unknown paths**: by default unrecognized API paths are passed through with a warning; when on they are rejected (may break an agent's auxiliary requests, so off by default).
- **Hit counters**: scrubbed requests / warning events / unknown paths.
- **Model catalog (read-only)**: library × gateway snapshot merged, showing protocol slices, Responses→Chat adapters, credential readiness, vision / thinking badges, context, and unavailability reasons. When a third-party tool treats the local gateway as a provider, **the model name must be one of the listed `kaya/<alias>` values** — a bare upstream name is rejected with HTTP 400. Edit models under the Models tab.
- **User token map**: fixed replacement rules — outbound "truth" becomes a "token", responses are restored as-is (e.g. `Zhang Wei` → `person-a`).

**Privacy statistics** (sub-tab at the top of the same page)

Shows categories, actions, restoration counts, and per-hit details. All rule records are shown by default: unreplaced risk records first, then rule-confidence order, with replaced records last. Actions distinguish outbound replacement from observed response restoration; an unreplaced match alone does not prove disclosure. Filter by category, action, agent, model route, rule type, and time window; search originals or tokens; aggregate repeated content with occurrence counts plus first/last-seen times. **Details contain matched originals and replacement tokens**, stored only locally, masked until a row is clicked, and never automatically added to LLM context or diagnostic exports. Expanded rows expose timestamps and record/request IDs and can open the original local JSONL. Per-request audit details are collapsed by default. The view reads the last 90 days with at most 5,000 active records; **Clear current view** or **Clear all records** both require confirmation. This is a separate local sensitive-data store, distinct from the content-free gateway scrub event `gateway-scrub` (the chat audit trail keeps scrubbed text).

**Network proxy (Settings → General)**

Configure a manual **HTTP / HTTPS / SOCKS5** proxy for kaya itself. Enter a hostname or IP and a port from 1–65535, with an optional username and password; SOCKS5 uses remote DNS. Passwords are write-only and stored only in the system keychain, never in localStorage; leaving the field blank preserves the existing password, and it can also be cleared explicitly. Direct-connection exceptions accept hostnames, IPs, and CIDR ranges; URLs, ports, paths, wildcards and malformed networks are rejected before saving. `localhost`, `127.0.0.1`, and `::1` always connect directly and cannot be removed. Unrecognized entries left by older versions are ignored individually and remain on the proxy path; Settings shows only the ignored count. Loading never rewrites the saved file, while the next explicit save writes the currently visible list.

A single endpoint is the default. With **Configure by target protocol** enabled, HTTP, HTTPS, and SOCKS5 endpoints can be stored at the same time, each with its own activation and authentication. Each target type prefers its dedicated endpoint; when that endpoint is off, SOCKS5 is tried next, then the other endpoint type (HTTP can carry HTTPS via CONNECT, and an HTTPS proxy can forward plain HTTP). At least one endpoint is required while the proxy is enabled; otherwise saving and startup are rejected.

**Proxy application scope**: models (Chat, provider OAuth), gateway upstream, account and sign-in, usage reporting (token usage, heartbeat, and their token refreshes), web search, IM channels, remote access, and updates can each choose proxy or explicit direct. Direct scopes never read the system proxy, PAC, or process environment; scopes left on proxy still fail closed when it is unavailable. Token refreshes initiated by a feature follow that feature's scope.

Once enabled, built-in Chat / BYOK / OpenAI Responses, remote model-gateway upstreams, OAuth, model discovery and health checks, Tavily, usage reporting, official update checks and updater downloads, and Feishu / Slack / WeCom / Remote REST and WSS traffic use the endpoint selected by the current proxy configuration, except for scopes explicitly set to direct. The audit judge uses the same exit. Disabling the proxy preserves the draft and explicitly selects direct connections without reading the system proxy, PAC, or process proxy environment. Requests left on proxy fail closed when it is unavailable or credential state is uncertain; they never silently fall back to a direct connection.

**Shell isolation**: this setting affects only kaya's own outbound requests. It does not write `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, or `NO_PROXY`, modify shell profiles, or inject PTY launch arguments. Terminal sessions, PTYs, AI `run_shell` subprocesses, and agents launched from them are unaffected; configure a proxy for terminal programs yourself when needed.

**General → other sections**

| Item | Description |
|---|---|
| Language | UI language |
| Shell | Shell path used by the PTY |
| Preserve terminal scrollback | Whether to keep each tab's scrollback after restart |
| Terminal font size | Terminal display size |
| Web Search | Enter the Tavily API key and enable web search |
| Scrubbing (Scrub Level) | User input is always aggressive; tool output defaults to aggressive and is adjustable here; gateway scrubbing is separate and defaults to strict (named secrets only) |

**Account**

View sign-in status, subscription / trial, refresh subscription, sign out; the Advanced section lets you paste a login token manually as a fallback.

The **Cloud Token usage** card is off by default and is enabled only after confirming the `kaya-usage-upload-v2` upload disclosure. It uploads only usage buckets aggregated by UTC hour or day (source, agent, normalized provider key, model id, token counts, request count) and the four device-registration fields (random device id, OS family, kaya version, consent version), never conversation or tool content. Turning it off stops further uploads from this machine but does not delete data already uploaded to the cloud. If an old-schema upload state cannot prove its historical identity, uploading stops and the old consent is cleared. A new consent **uploads only new usage by default**; the last 30 days require a separate explicit confirmation after previewing the scope, and the current bucket does not need to close first. A normal disable/re-enable on the same trusted account still backfills usage generated while disabled. Recognized account changes and revocations remain isolated from the old identity's history, while new usage in the current bucket starts syncing immediately. Desktop events belong to the bundle's deployment; new standalone CLI events stay in local statistics and are not uploaded. The status area shows the latest result, accepted bucket count, waiting time, and skip reason, displayed in the local timezone while buckets and cursors remain UTC.

**Separate upload state for CN and Intl**: CN and Intl installations on the same machine keep independent local upload state, consent, cursors, device identities, and history-backfill boundaries. An account change in one deployment is not mistaken for an account change in the other. Legacy shared state is adopted only after same-origin evidence and a real read-back succeed; uncertain identities never silently resume uploading or lift history isolation.

**macOS in-app update restart**: after installing an update, an independent helper waits for the current process to exit and then relaunches the app through LaunchServices, preventing the new version from running in the background without coming to the foreground.

**Audit**

View past sessions and run evaluations.

**Schedules**

Create chat turns that fire automatically on an interval or at a fixed time each day; toggle, delete, and cap the run count. See [Scheduled Tasks](#scheduled-tasks).

**Memory**

Review automatically stored memories, browse core / semantic / episodic entries, and consolidate or clean up manually. See [Memory](#memory).

**Logs & Debug**

View diagnostic logs and **manually** prune logs older than 7 days; dev builds also include the Debug section in this tab.

**About**

About is not a Settings tab. Open the standalone About dialog from the **system menu** (the kaya menu in the macOS menu bar → About kaya) to view the version, build, and license; **Check for Updates…** lives in the system menu too — when a new version is found, on macOS you can click **Download and Install** for a one-click update (the download is signature-verified and replaces the app in place, no DMG mounting or manual dragging; it takes effect after restart), or you can still download the installer in a browser and overwrite manually. The system menu also provides **IM Channels…** and **Keyboard Shortcuts…** shortcuts that jump straight to the corresponding Settings tabs.

---

## Keyboard Shortcuts

> **Windows hotkey mapping**: the tables below show the macOS ⌘-based hotkeys; on Windows they all map to **Ctrl+Shift** combos (e.g. ⌘T → **Ctrl+Shift+T** for a new tab, ⌘W → **Ctrl+Shift+W** to close, ⌘D → **Ctrl+Shift+D** to split), and ⌥⌘ arrow keys map to **Ctrl+Alt** arrows; one exception: the macOS horizontal split ⌘⇧D has no Windows counterpart — use **Ctrl+Shift+S** for a stacked split on Windows; everything else (Enter / Esc, etc.) is the same on both platforms.

**Tabs**

| Shortcut | Action |
|---|---|
| **⌘T** | New tab |
| **⌘W** | Close the current pane (when focus is in the terminal; with a single pane this closes the tab, and the last pane of the last tab closes kaya); close the entire kaya (when focus is elsewhere) |
| **⌘1** – **⌘9** | Switch to the Nth tab |
| **⌘⇧[** | Previous tab |
| **⌘⇧]** | Next tab |
| **⌥⌘←** / **⌥⌘→** | Reorder tabs (when focus is on the tab bar) |

**Split Panes**

| Shortcut | Action |
|---|---|
| **⌘D** | Split pane vertically (side by side) |
| **⌘⇧D** | Split pane horizontally (stacked) |
| **⌘⌥←→↑↓** | Move focus between panes |

**Focus**

| Shortcut | Action |
|---|---|
| **⌘⇧F** | Focus the last-active pane / exit focus (same as the ⛶ button) |

The chat panel also has a persistent ‹/› handle on its right edge: click ‹ to collapse the chat into a narrow rail so the terminal takes the full width, click › to expand it back (same view state as ⛶ / ⌘⇧F, just a different entry point).

**App**

| Shortcut | Action |
|---|---|
| **⌘,** (**Ctrl+,** on Windows) | Open Settings |

**Terminal Font**

| Shortcut | Action |
|---|---|
| **⌘=** / **⌘+** | Enlarge by 1px |
| **⌘-** | Shrink by 1px |
| **⌘0** | Reset to default size |

**Terminal Editing**

| Shortcut | Action |
|---|---|
| **⌘C** | Copy selection |
| **⌘V** | Paste |

> On macOS, **Ctrl + letter** key combos (such as Ctrl+W, Ctrl+A) are passed **directly to the terminal** for use by vim, readline, bash, and similar tools.

**Chat**

| Shortcut | Action |
|---|---|
| **Enter** | Send message (when Chinese IME is not active) |
| **Esc** | Interrupt the AI's current turn |

**About ⌘W**

The behavior of ⌘W depends on the current focus:

- Focus in the terminal: close the current pane (with a single pane this closes the tab; the last pane of the last tab closes kaya entirely)
- Focus elsewhere (chat input, settings, etc.): close the entire kaya

---

## FAQ

**Can the AI read my files?**

By default, no. Commands like `cat`, `head`, and `grep` are blocked by default. If you need the AI to read a file, place it under `~/.kaya/` and tell the AI the path. See [Let the AI Read a File](#let-the-ai-read-a-file).

**Will pasted keys be sent out?**

Keys matching recognized patterns are scrubbed before entering chat and logs, with a "scrubbed" UI marker. Detection covers common patterns such as PEM private keys and API keys, but cannot guarantee that every secret is recognized. Do not paste important keys into chat. Images bypass text scrubbing, so screenshots containing keys are not safe inputs either.

**Is my API key safe?**

Your API key is stored only in your local system keychain, and is sent only to the model endpoint you configured — it is never uploaded to Yunhou and never written to audit logs. It is **not** sent to any other third party. kaya does **not** promise "never uploaded" — any network request needs authentication, and authentication needs the key; that is the basic mechanism of network requests.

**What does the cloud model receive?**

Your selected model receives conversation text, tool results and applicable context, processed according to each path's protection rules. File content you deliberately provide and scrubbed terminal output can enter that context. Supported BYOK images are sent separately without text scrubbing. API keys authenticate the relevant service through headers, not conversation bodies. Other egress paths and exceptions are described under Privacy and Security Boundaries and in the privacy design.

**How do I use a local model?**

Choose **Ollama** as the provider: start `ollama serve` in a terminal first, then pick Ollama from the catalog in **⚙ Settings → Models → + Add Provider** (the endpoint is pre-filled with the local address; local endpoints need no API key). You can also pick "custom" and enter any local OpenAI-compatible endpoint (self-hosted vLLM / LM Studio). With fully local inference, neither chat nor tool output leaves your computer; if you are signed in, the usage heartbeat (uptime / platform / version only) still reports according to your settings and can be turned off in **⚙ Settings → Account**.

**What if the AI accidentally deletes my files?**

kaya's built-in AI `run_shell` cannot delete, move or directly edit your files; safe redirection and authorized config management have separate boundaries. A coding agent you run in the terminal has its own file permissions and confirmations: kaya's tool allowlist is not a global sandbox around that agent. The UI explains blocked operations.

**Why was a command denied?**

The AI's command may fall into one of these categories:

- Not on the allowlist; denied by default (try a different phrasing)
- Destructive command (such as `rm -rf`)
- Interactive program (such as `vim` or a REPL)
- System-modifying command (such as `brew install` or `sudo`)

The UI shows the specific reason.

**Why does macOS pop up a folder access permission?**

When the AI tries to read files under macOS-protected directories (Desktop, Downloads, Documents), macOS asks for permission. This is macOS's own privacy mechanism; all similar tools trigger it. See [Let the AI Read a File](#let-the-ai-read-a-file); the recommended way is to put files under `~/.kaya/` instead.

**How do I let the AI read a specific file?**

Put the file under `~/.kaya/`, then tell the AI the path in the chat. See [Let the AI Read a File](#let-the-ai-read-a-file).

**How do I recover a disconnected tab?**

Open **⚙ Settings** → the **Shell** section of the **General** tab, then click **Reconnect** to re-establish the PTY for that session.

**Why does connecting to the LLM also time out?**

The first connection to your LLM provider is bounded by a 30-second handshake timeout (triggered by network failure, bad API key, or provider rate-limiting). The UI surfaces a "LLM connection timed out" banner. Check your network and API key, then retry.

**How do I view the AI's past actions?**

Open **⚙ Settings** → **Audit**, pick a session, view the event stream or run an evaluation. See [Audit Records](#audit-records).

**Which platforms are supported?**

Both macOS and Windows versions are currently released:

- **macOS**: requires macOS 12 or later; one universal DMG covers both Apple Silicon and Intel
- **Windows**: x64, available as both NSIS (`.exe`) and MSI installers; the terminal is PowerShell (pwsh 7 preferred, Windows PowerShell 5.1 as fallback)

**Does the model gateway slow things down?**

It adds one loopback hop (`127.0.0.1`), not an internet round trip — usually milliseconds. The real cost is scrub level: strict only blocks named secrets automatically and is nearly free, balanced adds PII rules, while aggressive's high-entropy scan is noticeably heavier on large requests. If it feels slow, set the gateway back to strict.

**Why did my agent's model name become `kaya/xxx`?**

That is gateway form: the agent config holds an **alias** (`kaya/<slug>`) plus a friendly display name, and the gateway resolves it to the real library model when forwarding. This keeps the real provider / endpoint / key out of agent config. Aliases are **frozen once generated** (renaming does not change them), and old aliases resolve forever.

**My agent gets a 400 "model not in the gateway library" — what now?**

The request's model name is not a resolvable `kaya/<alias>`. Common causes: editing the agent config to a bare upstream name by hand, another tool's default model, or filling a bare name when a third-party tool first connects. Set it back to `kaya/<alias>` (see the catalog) or press **Re-attach**. The gateway **never** silently forwards unmanaged models — that is deliberate. The only compatibility exception is an explicitly globally taken-over Codex bound to an OpenAI ChatGPT subscription, where Codex may use its built-in bare model names.

**Does the gateway store my request content?**

No. Gateway audit is **content-free** (category counts, byte counts, actual route only). The only local details are **privacy statistics** — they hold the **matched fragment and its replacement token**, stored locally, masked by default, filterable and clearable, never fed into LLM context or diagnostic exports, and never leaving the machine.

**What's the difference between scrub levels?**

There are three **independent** levels:

- **Your input** (chat composer) = always **aggressive**, regardless of settings, and cannot be lowered.
- **Tool output** (gate 5 of the five-gate pipeline) = defaults to **aggressive**, adjustable under **Settings → General**.
- **Model gateway** = its own level, **strict** by default: named secrets only; ordinary PII is left unchanged. Balanced (named secrets and PII tokenized) and aggressive (tokenize high-entropy too, more false positives) are available.

The three chat-side levels mean:

- **aggressive**: recognizes the most categories of sensitive information, including ID numbers, phone numbers, emails, and credit card numbers
- **balanced**: recognizes common API keys, AWS credentials, tokens
- **strict**: recognizes only the most obvious key markers

Keep **aggressive** on the chat side; the gateway defaults to **strict** (named secrets only) to minimize false rewrites. Switch it to balanced or aggressive only when you know your traffic carries patterns the named rules miss.

---

## Troubleshooting

**Terminal Not Responding**

1. Press **Esc** to interrupt the AI's current turn
2. Open **⚙ Settings** → the **Shell** section of the **General** tab, click **Reconnect** to re-establish the PTY
3. If still unresponsive, try **⌘W** to close the current tab, then **⌘T** to open a new one

**AI Replies Are Slow**

- Check your network connection
- Thinking Mode adds latency (reasoning itself takes time)
- Switch to a faster model in Settings

**Tool Call Denied**

The UI shows the specific denial reason. Common cases:

- Command is destructive or interactive: try a different approach
- Command is outside the allowlist: try an equivalent tool
- Trying to read files outside `~/.kaya/`: move the file under `~/.kaya/`

**Paste Not Working**

Make sure focus is on the correct input: pasting into the terminal input goes to the terminal; pasting into the chat input goes to the AI.

**Session Lost**

Check whether the `~/.kaya/conversations/` directory has been cleaned. Old sessions should be auto-recovered from disk after restarting the app. If they still don't appear, you may need to clear the app's local state and start over.

**AI Stops Responding**

Streaming responses automatically time out after 30 seconds of silence; the UI shows a timeout notice. You can press **Esc** or click **⏹** to interrupt manually.

**macOS Folder Permission Popup**

Put the files to be read under `~/.kaya/`. Do not let the AI directly access protected directories like Desktop, Downloads, or Documents.

**Built-in Model Reports "Request Rejected by Upstream (502)"**

The upstream model service refused the request (context too long, content review, or quota are all possible). kaya first auto-compacts earlier conversation history and retries once in the background (a "history auto-compacted" notice appears in the message stream when this happens); if it still fails, the error card shows guidance for the specific cause plus a **"New session" button** — for context-length rejections, starting a new session is the fastest fix, and your terminal tabs and running agents are unaffected.

**The gateway reports "occupied" or won't start**

The dev and prod builds exclude each other: only the first to start holds the gateway, the other shows "occupied by <variant>" and refuses writes. Close the holder (usually the dev build) and enable again. If another program owns the port, the gateway automatically moves to the next free port — no manual edit needed.

**The gateway is on but an agent gets 401 / 403**

Check the agent detail page's "Connection status" first: missing credentials, OAuth not signed in / expired, or a disabled endpoint all show as "not connected + reason". Re-sign-in fixes an expired OAuth provider; when an API key is cleared, dependent agents leave the gateway form and re-attach automatically after you save a new key. If it still reports an auth error while "connected", use the provider's "Test connection" to tell "endpoint unreachable" apart from "credentials may be invalid".

**The gateway is on but an agent produces no output for a long time**

After 120 seconds (default, configurable) without upstream data the gateway aborts and sends one failure frame in the agent's protocol; the agent retries by itself — this is not a hang. If it happens often, check the network and the upstream service. Timeouts and broken streams are recorded in the App diagnostic log (zero content), and the counters are only exposed through the `gateway_get_status` command, not shown on the gateway page.

Discovered models inherit all enabled provider endpoints by default. This does not verify that a model works across protocols: provider catalogs may differ, and requests can still fail.

**Diagnostic Logs**

For other issues, open **⚙ Settings** → **Logs & Debug**, click "Open log folder" to view the log files under `~/.kaya/logs/`. These logs contain kaya's runtime diagnostic information (not your conversation content).

---

*Document version: v3.2.10 · Last updated: 2026-11-06 · Synced with the current product positioning and capabilities*
