# kaya User Guide

> **Mission Control for AI Coding Agents · Privacy First · Fast · Simple & Effective**

> [中文版](PRODUCT.md)

---


## Welcome to kaya

kaya is a macOS / Windows desktop app that brings an AI assistant directly into the terminal you use every day. Open it and you will see two panels: an AI chat panel on the left, and a real terminal on the right, just like Terminal or iTerm2. The two share the same shell session. Whatever you see in the terminal, the AI can also see, and you can ask the AI to explain, query, or help you operate.

kaya's goal is to give you AI assistance **without leaving your real shell**: you keep your zsh aliases, your scripts, your familiar shortcuts, and the AI does not trap you in a sandbox.

> Note: the AI does not take over your entire terminal. By default it only reads your terminal output; only when you **turn on the coding-mode toggle in the top bar**, explicitly allow it on the grant card in the chat, and that tab is running a supported AI CLI (Claude Code / Kimi Code / OpenCode / Codex) — can the AI type into that tab on your behalf. The coding-mode toggle is the master gate for "driving": while it is off, the chat side does not have the ability to send input to terminal AIs at all (the tool is not even registered); once on, each tab still requires its own grant-card approval. The grant is bound to the current process and expires when the process exits; you can revoke it anytime via the 🤖 badge on the tab. Your regular shell prompt never accepts AI input. Likewise, when the AI helps you edit a coding agent's config file (e.g. Claude Code's settings.json), every edit lands only after you explicitly approve it on a grant card, and the grant is scoped to the current conversation.

---

## What kaya Can Do

kaya is well suited for everyday tasks like these:

- **Code explanation**: paste an error, a shell script, or a regex and ask the AI what it does
- **Environment checks**: inspect your system info, disk usage, running processes, network ports, and so on
- **File queries**: list directory contents, count file sizes, find files by name pattern
- **Command construction**: let the AI help you compose `find` / `awk` / `sed` pipelines, then review before running
- **Log and error interpretation**: have the AI explain a block of error output and suggest possible causes

kaya does **not** do, and does not try to do, the following:

- Automatically rewriting source code on your Mac
- Running destructive operations without your confirmation (for example, deleting files)
- Running interactive programs in your terminal (such as `vim` or REPLs)
- Accessing the contents of arbitrary files on your Mac without your permission

Any command the AI suggests still needs to be **seen by you and executed by you** in your terminal. This is part of the design: the AI assists; you decide.

---

## Privacy and Security Boundaries

Privacy is the **first priority** of kaya. Below is the boundary you can expect by default.

**What the AI Cannot See by Default**

- The **contents** of any file on your computer: read commands like `cat`, `head`, `grep`, `less` are blocked by default
- Your computer's system environment variables: `env` and `printenv` are blocked by default
- Commands and output you type in your terminal yourself — this part is **entirely yours**, the AI does not intercept or review it

**What the AI Can Query by Default**

- **Metadata**: directory listings, file sizes, modification times, current user, system version, and so on
- The current output of your terminal (so you can ask the AI to explain an error)
- Your task list (read-only: the AI can view it but not modify it)
- Text you actively paste into the chat

**The `~/.kaya/` Trusted Directory**

If you need the AI to read the contents of a file, place the file under `~/.kaya/` and tell the AI the path. This is the **only** location where the AI can read file bodies. See [Let the AI Read a File](#let-the-ai-read-a-file) for details.

**Scrubbing Is a Defense, Not an Absolute Guarantee**

When you input text, when the AI reads a file, and when tool results come back, kaya automatically recognizes and replaces common keys, PEM private keys, ID numbers, phone numbers, credit card numbers, and other sensitive patterns. The built-in sensitive-data detection covers 30+ rule families across common scenarios. But kaya does **not** promise to recognize **every** possible secret — do not paste truly important keys into any LLM application.

**What Cloud Providers Receive**

When you send a message in the chat or let the AI call a tool, the scrubbed chat text and scrubbed tool results are sent to the **model source you selected in the top-bar model switcher**:

- **Built-in model (default, yunhou `/chat` proxy)**: scrubbed content is sent over your login token (JWT) to the **yunhou first-party server**, which answers on your behalf with yunhou's DeepSeek key. Before sending, the kaya client trims to the server's limits (message count / per-message bytes / total bytes — the oldest history is dropped first). Your login token (refresh_token) lives only in the system keychain (macOS Keychain / Windows Credential Manager).
- **OpenAI compatible (your own key)**: scrubbed content goes to the LLM provider you chose (DeepSeek / OpenAI / OpenRouter / etc.), authenticated with your API key.
- **Ollama (local)**: nothing is sent (see below).

kaya does not send your content to any third party beyond the paths above. Local UI state, sessions, audit logs, and application logs all stay on your computer.

**Usage Statistics (Optional)**

When signed in, kaya collects usage statistics that contain no conversation content (active days, usage duration, platform and version distribution) to help us understand overall product usage (since v2.1.14, on by default). It **never** includes conversation content, file content, or personal identity information, and can be turned off anytime in **⚙ Settings → Account**; nothing is reported while signed out.

**How Your API Key Is Used**

The API key you enter in Settings is **stored only in your local system keychain** (macOS Keychain / Windows Credential Manager). It is sent only to the LLM provider you selected, in the authentication header of each request, for that provider's authentication. kaya does **not** send your API key to any other party. kaya does **not** promise "never uploaded" — any network request needs authentication, and authentication needs the key, that is the basic mechanism of network requests.

**Offline Inference: Ollama**

If you choose Ollama as the provider, all inference happens **entirely on your computer**, with **no outbound requests**. This is suitable for the most privacy-sensitive scenarios.

**Five Layers of Protection, in Brief**

In short, everything from your input to the AI's output goes through:

1. **Input guard**: pasted keys and long text are scrubbed and truncated
2. **Command policy**: executable commands are restricted by an allowlist; unknown commands are denied by default
3. **Safety check**: destructive, interactive, and system-modifying commands are blocked
4. **Isolated execution**: AI tool calls run in independent subprocesses and never touch your terminal
5. **Output scrub**: tool output is scrubbed again before entering the chat

---

## Install kaya

**Primary Path: Download the Installer from the Official Site**

- **macOS**: open the [kaya download page](https://yunhou.ai/download) and download `kaya-x.y.z-macos-universal.dmg` (universal for Apple Silicon and Intel):
  1. Double-click the DMG to mount it; a window showing `kaya.app` will appear
  2. Drag `kaya.app` into the `/Applications` folder
  3. Open kaya from Launchpad or `/Applications`
- **Windows (x64)**: from the same download page, download `kaya-x.y.z-windows-x64.exe` (nsis, recommended) or `.msi`, then run the installer

**First Launch**

If macOS shows "cannot be opened because Apple cannot check it for malicious software":

1. Find `/Applications/kaya.app` in Finder
2. Right-click and choose "Open"
3. Click "Open" again in the dialog

After this, you can double-click it normally.

> Note: the official user guide recommends installer downloads only. Pre-compiled CLI releases are currently paused; advanced users who need the CLI should refer to the separate CLI documentation.

---

## First-time Setup

After opening kaya for the first time, do the following.

**Option A (default, no API key): built-in model + WeChat sign-in**

By default kaya uses the **built-in model** (the yunhou `/chat` proxy — the server answers with yunhou's DeepSeek key), so you don't need to apply for an API key. You only need to sign in to your yunhou account:

1. Click **⚙ Settings**, or click **Sign in with WeChat** in the chat window (the account icon / sign-in button in the top bar)
2. Your system browser opens a WeChat scan page → scan → you're redirected back to kaya (the token is written to the system keychain)
3. Return to the chat window and ask away. Subscription / trial status is under **⚙ Settings → Account**

> Two-tier feature gating (since v2.1.5): **signing in** alone lets you chat for free with your own models (BYOK); a **paid subscription** unlocks the built-in model, coding mode, IM channels, and remote access.

**Option B (your own API key): OpenAI compatible**

If you want to use your own LLM key:

1. Open **⚙ Settings → the Models tab**, and add a model configuration in the model library (My Models)
2. Pick a **Provider** preset — 11 in total (since v2.1.14):
   - **DeepSeek**: commonly used in mainland China; default model `deepseek-flash`
   - **Kimi (Moonshot)**, **GLM (Zhipu)**, **MiniMax**, **Qwen (DashScope)**, **SiliconFlow**, **OpenCode Zen**
   - **OpenAI**: international, requires overseas access
   - **OpenRouter**: aggregates many models; switch models under one key
   - **Ollama**: local inference; requires `ollama serve` to be running
   - **Custom (OpenAI compatible)**: bring any OpenAI-compatible endpoint with its Base URL and model
3. Enter your **API Key**, obtained from your provider's website; the key lives only in the system keychain
4. Check the **Base URL**: it is auto-filled when you pick a preset; usually you do not need to change it
5. Select a **Model**: the defaults follow each vendor's current lineup and work for most tasks; thinking-capable models automatically get the right reasoning parameters
6. Optionally turn on **Thinking Mode**: lets the AI show its reasoning before answering
7. Click Save, then return to the chat

You can save **multiple model configurations (profiles)** and switch between the built-in model and your own models with one click from the **model switcher in the top bar** (since v2.1.8). Once configured, kaya remembers your settings — you do not need to reconfigure next time.

---

## Your First Conversation

After configuration, type this in the chat input at the bottom right:

```
Show the current system version and CPU architecture
```

Press **Enter** to send. You will see:

1. The AI first shows a collapsible reasoning block (if Thinking Mode is on)
2. Then a **tool card** appears, showing which tool the AI called and the arguments
3. The card status changes from **running** to **done**; expand it to see the full stdout / stderr
4. The AI summarizes the result in natural language

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

---

## Slash Commands in Chat

Type **`/`** in the chat input to summon the command menu (pick with the keyboard or mouse to autocomplete); `/help` lists all commands anytime. The built-in commands:

- **`/agents`**: lists the currently running companion agents (kimi / claude / opencode / codex, etc.). Click a card to bind it as this session's forward target — the binding is per-session and cleared on a new conversation
- **`/new`**: start a new conversation (same entry point as the New button in the top bar)
- **`/sessions`**: lists past sessions from the audit trail; click a card to view its read-only history
- **`/help`**: lists every command with a description

**Mention an agent directly with `@` (since v2.3.1)**: type `@` in the chat input to summon the list of currently running coding agents (keep typing to filter — matches by name or project label); picking one binds it as this session's forward target and autocompletes the `@instance ID` into the input box, so you can see exactly who the message will go to — subsequent messages route straight to that agent without opening `/agents` first. It is the same binding channel as the sidebar cards and `/agents`.

Input that starts with `/` but isn't a built-in command (such as `/auto`) does not error out: if the current session has a bound agent, kaya writes the original text into that agent's terminal input line and presses Enter for you. Before writing, it fails closed on several checks (the pane is still alive, the foreground process is still the bound agent, and it is not waiting at a menu); if any check fails, nothing is written. These forwards **do not consume an LLM turn**; each forward appears as a card in the message stream and is recorded in the audit log. When you re-bind across tabs, kaya injects a one-time context note to the newly bound agent so it knows what came before.

---

## Task List and Task Queue

kaya ships a lightweight task list to help you and the AI track what needs doing:

- The todo toggle in the top bar opens the **Task panel**, where you can view, check off, and manage current tasks
- You can also manage it in natural language right in the chat: ask the AI to **extract** tasks from the conversation, or to **view / add / edit / delete / clear** the list
- On the AI side there is a single **read-only** tool (viewing the task list) to align on context — it cannot modify the list for you
- Tasks you're not doing yet can be **archived** (since v2.3.4): moved into a collapsible archive section at the bottom of the panel, restorable anytime; archived tasks do not appear in the AI-visible list and do not take up numbering

**Task Queue (Queue, handoffs between agents, since v2.3.4)**

The task list tracks long-term todos; the **task queue** tracks what needs to run right now — the two live in separate Tasks / Queue tabs of the task panel. Typical scenario: one agent finishes its current work and the next step needs **another agent** — put the task (with handoff context and acceptance criteria) into the queue:

- Once the target agent goes idle, kaya automatically opens a chat turn in which the AI writes the handoff based on the previous agent's outcome and dispatches it; dispatch reuses the two authorization gates (the coding-mode master switch + that tab's grant card). Tasks without a grant show "waiting for authorization"; you can also click "Dispatch now" to trigger manually
- When the run finishes, kaya runs a **verification round** against the acceptance criteria and judges pass / fail / inconclusive: a pass marks the task done; a failure keeps the reason, and you can retry with one click or ask the AI for help; an inconclusive result is automatically re-verified, and repeated uncertainty is marked as failed
- Queue contents stay only on this machine and survive restarts; a task interrupted mid-dispatch safely returns to the queue
- Entry points: the form at the top of the Queue tab, "Send to queue" on a task-list entry, or automatic creation when the AI detects a handoff in chat

**Cross-agent To Do List (since v2.3.7)**: there is also a collapsible To Do List area above the chat input — when the AI finds that a follow-up task needs another agent, it writes the task here; once the target agent goes idle, kaya first asks you "run it now?" and only dispatches on your explicit yes; if the agent has already finished the task on its own, kaya marks it done against on-screen evidence and tells you, without re-dispatching (v2.3.9). It never mixes with your familiar task list.

**Scheduled Tasks (Schedules, since v2.1.12)**

Create automation tasks in **⚙ Settings → Schedules**: have kaya automatically start an AI chat turn for you every N minutes or at a fixed time each day (for example, check build status every 10 minutes, summarize terminal output every morning at 9). Tasks only fire while the app is running; you can set a maximum run count, and the task disables itself once the limit is reached (since v2.1.13). You can also ask the AI in chat to list / add / delete / start / stop scheduled tasks — every change pops a one-time grant card and takes effect only after you approve.

---

## Terminal and Tabs

**Basic Terminal Operations**

The right panel is your real shell — on macOS a login zsh (loads `.zshrc`; aliases and environment variables behave exactly like your native terminal), on Windows PowerShell (prefers pwsh 7, falls back to the built-in Windows PowerShell 5.1; git-bash users automatically keep their original shell). Common actions:

- Type a command and press **Enter** to run it
- Select text with the mouse, press **⌘C** to copy, **⌘V** to paste
- **⌘A** to select the current line
- Scroll up with the scrollbar or trackpad to view past output

Commands you type in the terminal are **entirely yours**; the AI does not intercept or review them — that is your responsibility.

**Multiple Tabs**

kaya supports opening multiple terminals at once; each tab is an independent shell session:

| Action | Shortcut |
|---|---|
| New tab | **⌘T** |
| Close the current tab (when focus is in the terminal) | **⌘W** or the × on the tab |
| Switch to the Nth tab | **⌘1** – **⌘9** |
| Previous tab | **⌘⇧[** |
| Next tab | **⌘⇧]** |
| Reorder tabs (when focus is on the tab bar) | **⌥⌘←** / **⌥⌘→** |

Each tab has its own working directory, command history, and scrollback. When a tab is closed, its command history is automatically merged into the global history (on macOS into `~/.zsh_history`; on Windows into kaya's own global history file — your PSReadLine history is not touched).

**Tab renaming (since v2.1.12)**: double-click a tab title to rename it, making parallel sessions easy to tell apart; duplicate names are rejected with a prompt, and names persist across restarts.

> Windows shortcut mapping: the tables below show the macOS ⌘-family shortcuts; on Windows they uniformly map to the **Ctrl+Shift** family (e.g. ⌘T → Ctrl+Shift+T, ⌘D → Ctrl+Shift+D), ⌘⌥ arrow keys map to **Ctrl+Alt** arrow keys, and Enter / Esc are the same on both platforms.

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
- Note: coding-mode auto turns are sent to your selected model over the same path as your manual questions; with the toggle off, nothing is triggered at all

**Running-Agents Sidebar (since v2.3.0)**

With coding mode on, a collapsible **running-agents sidebar** sits at the left edge of the chat panel, giving an overview of every running coding agent:

- Each card shows: the instance number (e.g. `kimi-code #2`, a global sequence across agent types), its current state (waiting for you / idle / running), and its tab and working directory
- **Status line** (since v2.3.5): current model, thinking level, permission mode, context usage — kimi code shows all four; claude code shows the permission mode; codex shows the model; opencode is not supported yet. Permission modes are shown in plain words ("always ask / ask when needed / never ask")
- **Click a card**: switch to that tab + bind the agent as the current conversation's forward target + focus the input box — then direct it right from chat; kaya also has the AI glance at the screen and brief you on the agent's latest progress (repeat clicks within 10 seconds do not re-report)
- Cards in the "waiting for you" state **list the agent's option buttons inline** — answer with one click, no tab switch needed
- **Project labels** (since v2.3.4): each tab can edit its own project label (click the project line on the card); with multiple projects in parallel you can tell at a glance which agent is doing what, and the `@` list can match by project name; since v2.3.7 labels are per-pane
- Drag the right-edge handle to resize freely; on narrow windows it becomes an overlay drawer (click the backdrop to close); in terminal fullscreen (⛶ / ⌘⇧F) the sidebar stays docked at the window's left edge; the chat panel can also be collapsed into a narrow strip via the ‹/› handle on its right edge, yielding width to the terminal (since v2.3.5)

**Agents Panel**

Open **Agents** from the app menu (or click the Agents icon on the right side of the top bar). Installing, updating, and model configuration can also be done by asking kaya in chat (e.g. "install Claude Code for me") — both paths run the same engine:

- **Background install / update** of coding agents: kaya automatically tries npm > brew > the official script in turn, with live progress shown on the card; if one method fails it **automatically falls through to the next**, and only reports an error when all methods have failed (listing each method's own failure reason), keeping "Install in terminal" as a manual fallback
  - If the npm method hits a network-class failure, kaya retries **once** via the npmmirror registry; each method's single attempt has a **10-minute overall timeout** — if nothing seems to move, check the output tail (last 50 lines) on the panel card to tell slow from stuck
  - Uninstall: only npm/brew-managed installs are supported (with an ownership check — an install not managed by that package manager is never touched), and only the program itself is removed, keeping config and history; script-installed (official script) agents must be removed manually per the official docs
  - **Custom command names** (since v2.3.0): if you renamed a coding agent's CLI command (e.g. you use `myclaude` instead of `claude`), add those names in the "Command names" field on the agent's card (comma-separated); install detection, foreground-process recognition, and the launch command all accept both the original and your custom names
- **Unified model library** (since v2.2.0 / v2.2.1): model configuration is maintained centrally in **⚙ Settings → the Models tab** — the same library as the chat side's "My Models"; the Agents panel only **binds**: expand an agent's model list and click **Use** to write the entry into that agent's own config file (a `.bak` backup is kept), or **Unbind** anytime to let it fall back to its own defaults
  - One entry can carry three protocol endpoints — **chat / Anthropic / OpenAI**: Claude Code uses the Anthropic endpoint; Kimi Code / OpenCode / Codex use the OpenAI endpoint (falling back to the chat endpoint when it is empty). The OpenAI endpoint can also declare a **Wire capability** (Chat Completions / Responses / Both, since v2.2.4) — Codex officially supports only the Responses API and can only bind entries declared for Responses
  - **API keys live only in the system keychain** (macOS Keychain / Windows Credential Manager), never again in plaintext config files; scattered configurations from older versions are migrated and merged automatically on the first launch after upgrading
  - Editing or deleting an entry **automatically syncs** to all bound agents (no second Use click needed); after deleting an entry the agent falls back to its own defaults and no plaintext key remains in its config file. Exception: handwritten configs kaya never claimed (no kaya write marker) are not cleaned up automatically — click "Use" once for that agent to converge, or manually restore the neighboring `.bak`
  - **Codex special case**: to coexist with the ChatGPT desktop app, kaya writes Codex configuration to a dedicated profile (`~/.codex/kaya.config.toml`), effective only for Codex launched from kaya (or `codex --profile kaya`); a bare `codex` never sees this configuration
  - When kaya edits a config from chat, a grant card asks for your confirmation before anything hits disk; secret values are never echoed back into the chat; the AI can read configs to help you troubleshoot, but it **cannot** change the model name, endpoint, or key (since v2.2.1)
  - To roll back, manually restore the `.bak` backup written before the change (e.g. `~/.claude/settings.json.bak`)
- Over IM channels (Feishu / Slack / WeCom): only config read/edit is available (grant cards included); status checks, install, and update are not available there

---

## Let the AI Read a File

By default the AI cannot read files on your Mac. If you want the AI to help you analyze some code, view a log, or explain a configuration file, you need to first place the file in the trusted directory.

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

- `cat` / `bat` / `view`: read the entire file
- `head` / `tail`: read the beginning or end
- `grep` / `awk` / `sed -n`: search file content
- `wc` / `sort` / `uniq`: count and sort
- `strings` / `xxd` / `hexdump`: view binary content

**Disallowed Operations**

Even under `~/.kaya/`, the following remain blocked:

- Modifying files (`sed -i`, redirection, `mv`, etc.)
- Deleting files
- Executing file contents

The AI can **read, not modify**. This is intentional.

**Downloading Files**

The AI can also download files from the network into `~/.kaya/` (for example `curl -o ~/.kaya/...`). Dangerous file types (such as `.sh`, `.exe`, `.dmg`) are blocked.

**Reminders**

- Do not put keys into `~/.kaya/` just because there is scrubbing, hoping "the AI will look at them"
- Delete files under `~/.kaya/` promptly when no longer needed
- Symbolic links pointing outside `~/.kaya/` are not recognized as trusted

---

## Web Search

kaya can let the AI search the web through Tavily. Tavily is optional and disabled by default.

**Enable**

1. Sign up at [tavily.com](https://tavily.com) and obtain an API key
2. In kaya, open **⚙ Settings** → **Web Search**
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

Tavily is the **only** feature that sends your (scrubbed) question text to a third party. What Tavily receives is only the text of your question to the AI — not your files, not your API key. **Disable Tavily to close this outbound path**.

**Trustworthiness Reminder**

Search results are **untrusted input from the internet** — they may be outdated, incorrect, or biased. The AI's summary still needs you to verify key facts. Enabling Tavily does not bypass kaya's content protection: page content goes through scrubbing before entering the model context.

---

## Use kaya via Feishu / Slack / WeCom

Since v2.1.0, kaya can bridge IM messages to your machine: message the bot in Feishu, Slack (since v2.3.6), or WeCom (Enterprise WeChat) and your local kaya LLM handles it and **streams a reply** — no need to have the kaya window open.

**How to enable**

1. Open **⚙ Settings** → the **IM Channels** tab (also reachable directly from the system menu: kaya → **IM Channels…**) and pick a platform — **the WeCom entry is currently greyed out as "(coming soon)"** (existing WeCom configs are preserved and will be restored in a later version); Feishu and Slack are open for setup
2. **Feishu — recommended: click "Connect via Feishu QR"** (only Feishu offers a QR code): scan the QR code with the Feishu mobile app and confirm — kaya automatically creates the app and fills in the App ID plus the creator's open_id in the allowlist; the App Secret goes straight into the **system keychain** (macOS Keychain / Windows Credential Manager), never to disk, never echoed back
   - **Slack manual configuration** (no QR code): create an app at [api.slack.com](https://api.slack.com/apps) → enable **Socket Mode** and generate an App-Level Token (`xapp-` prefix) → add the OAuth scope `chat:write` → subscribe to the `message.channels` / `message.groups` / `message.im` / `message.mpim` events → install to the workspace to get the Bot Token (`xoxb-` prefix) → paste both tokens into Settings (likewise stored only in the system keychain)
   - Feishu manual configuration (fallback): create an app yourself on the [Feishu Open Platform](https://open.feishu.cn), then enter the App ID + App Secret
3. Configure the **admission allowlist**: one user ID per line (Feishu `open_id`, starting with `ou_`; Slack member ID, starting with `U`; WeCom member userid); Feishu QR provisioning locks it to the creator automatically
4. Click **Start channel**; once the status shows "running", message the bot from your IM. **If a permission error appears**, complete the permissions for the app in the corresponding platform console (Feishu: enable the im:message scopes and publish a version)

**Key points**

- **Direct long connection to the platform**: Feishu official WebSocket, WeCom official openws channel, Slack Socket Mode WSS — **no public IP or callback server needed**
- **Allowlist denies strangers by default**: messages from users not on the list are dropped before ever reaching the LLM; an empty list prevents startup
- **Same privacy pipeline**: IM messages go through the same five-gate pipeline as desktop chat; IM turns use the same full tool table as the desktop (terminal read / task list included), while `terminal_write` is still governed by the coding-mode master switch — that switch is an atomic flag shared between desktop and channel; toggling it on the desktop is pushed to the channel immediately via `channel_set_coding_mode`, taking effect from the next IM turn
- **Desktop → IM mirroring (opt-in, off by default)**: once enabled, the desktop chat stream is pushed in real time to your chosen IM chat, and your own inputs are echoed as "你:…"-prefixed text; IM replies also appear in the desktop chat panel. Starting a new desktop session migrates the mirror automatically and pushes a "new session started" notice. The toggle lives in two places — the "Mirror desktop chat to Feishu" checkbox in Settings and the mirror icon next to coding in the top bar; both are the same switch and take effect immediately
- **Feishu / Slack authorization cards**: when an IM turn needs terminal_write / agent-config approval, the IM shows an interactive card with Approve / Deny buttons (Block Kit buttons on Slack); decisions share the same grant store as desktop approval cards (mutually recognized); the operator must be on the admission allowlist. Once approved, kaya **automatically runs a follow-up turn** asking the AI to retry the paused operation — no need to wake it up with another message
- **Proactive agent-status notifications**: with mirroring on, the coding agent's waiting-for-input / idle-done / online / exited transitions are pushed to the mirrored chat (Chinese or English per the channel locale, with per-event cooldown to avoid flooding)
- Sessions and audit land in `~/.kaya/` as usual; the LLM uses the model source selected in the top-bar model switcher
- **Honest caveat**: WeCom mirror echoes and proactive notifications depend on an active callback registration and may be silently dropped in pure-desktop scenarios (no IM turn in flight); only Feishu was verified on a real device in this round — Slack has not yet been validated against a real workspace

See the "IM channel entry" and "IM channels & mirroring" sections of [docs/PRIVACY.en.md](./PRIVACY.en.md) for details.

---

## Remote Access (browser, since v2.3.2)

Keep working even when you're away from your computer: view and operate this machine's kaya from a **browser** — conversations, the running agents' status lines (model / thinking level / permission mode / context usage) and working directories, and approval prompts all stay in real-time sync with the desktop.

**How to enable**

1. Open **⚙ Settings → the Remote Access tab** (requires signing in to your yunhou account with an **active subscription**; otherwise the switch is greyed out with guidance)
2. Turn on the "Enable remote access" switch; once the status shows "running" and the connection state shows "online", you're set
3. In the browser, sign in to the yunhou website and open the "Remote Control" page to reach this device

**Privacy boundaries**

- **Off by default, explicitly opt-in**; turning it off disconnects immediately
- **In-memory relay only**: conversation content transits the yunhou server in memory and is **never persisted**
- **API keys never transit the relay** — this machine still connects to model providers directly; everything is TLS-encrypted
- Remote turns go through the **same five gates** as the desktop; write operations like `terminal_write` share the same grant store with the desktop (mutually recognized)
- The device name is customizable (≤64 characters) to tell multiple machines apart; the device identity lives at `~/.kaya/remote/device.json` (permissions 0o600); when the subscription expires, remote access stops automatically and says so

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

Open **⚙ Settings → the Models tab**, find **Thinking Mode**, and turn it on.

**Switching the Reasoning Model**

kaya's Settings has a **Reasoning Model** field.

> **Important**: For **chat requests**, the **Reasoning Model** field is currently **display-only** — it does not switch the actual model ID in the AI's request. The field is currently used only by the **AI judge** in audit evaluations (to pick the judge model).

To switch which model the AI uses, change the **Model** field in Settings. When Thinking Mode is enabled, kaya automatically adapts reasoning parameters per the selected provider's rules (for example, DeepSeek injects a `reasoning_effort` parameter) — **you do not** need to switch to a separate "reasoning-only model".

If you see the Reasoning Model field, treat it as a **reserved** field for now; its value does not affect chat requests.

---

## Sessions and History

**Session Persistence**

kaya automatically saves your conversations. Three layers stay in sync:

- UI messages (the chat panel) are saved in the app's local state
- LLM context (for the AI to review) is saved in `~/.kaya/conversations/`
- Audit logs (the detailed event stream) are saved in `~/.kaya/audit/`

After refreshing the app or restarting your Mac, conversations and settings are restored automatically.

**New Session**

Click the **↻ New Session** button in the top right:

1. Stops any turn in progress
2. Clears the current UI messages
3. Starts a fresh session context
4. Begins a brand-new conversation

**View Past Sessions**

The **New Session** button is a plain icon button with no session list next to it. There are two ways to view history: use the **`/sessions`** command in the chat input to list past sessions and click a card to view them (**read-only** — you cannot resume the context and keep chatting); or open **⚙ Settings → Audit** for the full event stream.

**Clear Data**

kaya has no per-session delete UI. To clear all local data, run in the terminal:

```bash
rm -rf ~/.kaya/
```

**Tabs and Exit Behavior**

- **Close tab** (**⌘W**): confirmation dialog appears; on confirm, the tab closes and its shell history merges into `~/.zsh_history`
- **Close the entire kaya** (red ×): confirmation dialog appears; on confirm, all alive tabs' shell history merges into `~/.zsh_history`, then kaya exits
- **⌘Q**: no confirmation dialog; kaya first tries to merge local history into `~/.zsh_history`, then exits. Where possible, prefer the red × over ⌘Q

---

## Audit Records

kaya automatically records a complete audit log for every session. You can replay what the AI did at any time.

**Open the Audit Page**

**⚙ Settings** → **Audit**.

**Sessions List**

The left side lists all sessions, sorted by time. Click any session to view its event stream.

**Event Types**

- Session start / end
- Each of your inputs (with scrub results and original byte count)
- Each AI tool-call decision (why allowed or denied)
- Tool-call completion (with stdout, stderr, exit code)
- Each AI reasoning chunk
- Error events

**Run Evaluation**

Click **Evaluate** on a session; kaya performs three passes:

1. **Rule replay**: re-runs current policy rules against all recorded tool-call decisions. Any deviation from current rules is flagged
2. **Anomaly heuristics**: checks 5 anomaly patterns — "file read without scrub", "network output without scrub", "environment-variable output without scrub", "tool call missing preceding reasoning chunk", "large Tavily output without scrub"
3. **AI judge**: an independent AI (always in thinking mode) gives the final review of the entire session

**Audit Logs vs App Logs**

- **Audit logs**: per-session LLM-related events, used for replay
- **App logs**: cross-session diagnostic logs, used to troubleshoot kaya itself

The two are independent; do not confuse them.

---

## Settings Reference

Open **⚙ Settings**. The tabs are: **General / Models / Audit / App Logs / Shortcuts / Account / IM Channels / Remote Access / Schedules** (dev mode also has Debug). **Web Search, Shell, Language, Scrubbing** are all sections inside the **General** tab, not standalone tabs.

**Models**

The unified model library — the **same library** as the chat side's "My Models" and the Agents panel's model library; add / edit / delete model entries here. Binding a model to a coding agent is done in the Agents panel via "Use".

| Item | Description |
|---|---|
| Provider | LLM provider preset (DeepSeek / Kimi / GLM / MiniMax / Qwen / SiliconFlow / OpenCode Zen / OpenAI / OpenRouter / Ollama / Custom) |
| API Key | Provider key, stored only in the system keychain |
| Base URL | API endpoint; auto-filled when you pick a preset |
| Model | Main chat model |
| Protocol endpoints | One entry can carry three endpoints — **chat / Anthropic / OpenAI**; the OpenAI endpoint can declare a Wire capability (Chat Completions / Responses / Both) |
| Reasoning Model | Display field (currently used only by the AI judge in audit evaluations — see Thinking Mode) |
| Thinking Mode | Enable thinking mode |

> Note: **Max Context Messages** controls how much history is sent to the model (keeps the most recent N messages; the oldest are dropped first); each conversation persists at most 200 messages.

**General → other sections**

| Item | Description |
|---|---|
| Language | UI language |
| Shell | Shell path used by the PTY |
| Preserve terminal scrollback | Whether to keep each tab's scrollback after restart |
| Terminal font size | Terminal display size |
| Web Search | Enter the Tavily API key and enable web search |
| Scrub Level | Scrubbing strength (aggressive / balanced / strict); default aggressive; your input text is always scrubbed at the aggressive level |

**IM Channels**

Bridge Feishu / Slack / WeCom messages to this machine's kaya: configure platform credentials (stored in the system keychain) and the admission allowlist, start / stop channels, and view their running status. See [Use kaya via Feishu / Slack / WeCom](#use-kaya-via-feishu--slack--wecom).

**Remote Access**

The switch and connection state for browser-based remote access to this machine's kaya (off by default; requires an active subscription). See [Remote Access (browser)](#remote-access-browser-since-v232).

**Schedules**

Scheduled tasks: automatically start an AI chat turn every N minutes or at a fixed time each day, with an optional maximum run count. See [Task List and Task Queue](#task-list-and-task-queue).

**Account**

View sign-in status, subscription / trial, refresh subscription, sign out; the Advanced section lets you paste a login token manually as a fallback. The **usage statistics** toggle (active days, usage duration, platform and version distribution; no conversation content) also lives here — on by default, can be turned off anytime; nothing is reported while signed out.

**Audit**

View past sessions and run evaluations; click a session card to expand its event-by-event timeline (since v2.3.3).

**App Logs**

View diagnostic logs and **manually** prune logs older than 7 days.

**About**

About is not a Settings tab. Open the standalone About dialog from the **system menu** (the kaya menu in the macOS menu bar → About kaya) to view the version, build, and license; **Check for Updates…** lives in the system menu too. The system menu also provides **IM Channels…** and **Keyboard Shortcuts…** shortcuts that jump straight to the corresponding Settings tabs.

---

## Keyboard Shortcuts

**Tabs**

| Shortcut | Action |
|---|---|
| **⌘T** | New tab |
| **⌘W** | Close the current tab (when focus is in the terminal); close the entire kaya (when focus is elsewhere) |
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
| **⌘A** | Select all |

> On macOS, **Ctrl + letter** key combos (such as Ctrl+W, Ctrl+A) are passed **directly to the terminal** for use by vim, readline, bash, and similar tools.

**Chat**

| Shortcut | Action |
|---|---|
| **Enter** | Send message (when Chinese IME is not active) |
| **Esc** | Interrupt the AI's current turn |

**About ⌘W**

The behavior of ⌘W depends on the current focus:

- Focus in the terminal: close the current tab
- Focus elsewhere (chat input, settings, etc.): close the entire kaya

---

## FAQ

**Can the AI read my files?**

By default, no. Commands like `cat`, `head`, and `grep` are blocked by default. If you need the AI to read a file, place it under `~/.kaya/` and tell the AI the path. See [Let the AI Read a File](#let-the-ai-read-a-file).

**Will pasted keys be sent out?**

No. Keys you input are automatically scrubbed inside kaya (PEM private keys as whole blocks, API keys, ID numbers, phone numbers, emails, credit card numbers, and other common patterns). Only the scrubbed text enters the chat and the logs. Replaced parts are marked "scrubbed" in the UI.

**Is my API key safe?**

Your API key is stored only in your local system keychain (macOS Keychain / Windows Credential Manager). It is sent only to the LLM provider you selected, in the authentication header of each request, for that provider's authentication. It is **not** sent to any other third party, and it is never written to the audit log. kaya does **not** promise "never uploaded" — any network request needs authentication, and authentication needs the key; that is the basic mechanism of network requests.

**What does the cloud model receive?**

The cloud provider receives: your (scrubbed) chat text and the (scrubbed) tool results from the AI's tool calls. It does **not** receive: other local files, your API key value, or your environment variables. The provider is the one you selected in Settings (DeepSeek, OpenAI, etc.).

**How do I use a local model?**

Choose Ollama as the provider, make sure `ollama serve` is running in a terminal first, and enter the local URL. All inference runs locally; no data leaves your Mac.

**What if the AI accidentally deletes my files?**

The AI has no permission to modify or delete files on your Mac. Deletion commands (`rm`, `sudo`) and most write/rename commands are blocked by default. If the AI tries to run a blocked command, the UI shows the specific reason; you can rephrase or use a different approach.

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

macOS (universal dmg for Apple Silicon and Intel) and Windows x64 (msi / nsis installers; nsis recommended). Installers are published on the [kaya download page](https://yunhou.ai/download).

**What's the difference between scrub levels?**

- **aggressive** (default): recognizes the most categories of sensitive information, including ID numbers, phone numbers, emails, and credit card numbers
- **balanced**: recognizes common API keys, AWS credentials, tokens
- **strict**: recognizes only the most obvious key markers

Recommended: keep the default **aggressive**. **Note**: your input text is always scrubbed at the aggressive level regardless of this setting.

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

**Diagnostic Logs**

For other issues, open **⚙ Settings** → **App Logs**, click "Open log folder" to view the log files under `~/.kaya/logs/`. These logs contain kaya's runtime diagnostic information (not your conversation content).

---

*Document version: v1.5.0 · Last updated: 2026-09-23*
