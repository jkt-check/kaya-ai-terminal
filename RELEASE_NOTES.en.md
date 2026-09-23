# Release Notes

> User-facing release notes, sorted by version in descending order. Each version's new features, improvements, and fixes are described for end users, without internal implementation details.
> Maintenance convention: update this file on every release (the website consumes it as the release notes); security fixes describe only the impact and upgrade advice, never the exploitation details. Internal engineering records live in `CHANGELOG.md`.
> This is the English mirror of `RELEASE_NOTES.md` — keep both in sync.

## v2.3.9 (2026-09-23)

**🚀 Improvements**

- **To Do List decision loop, closed: kaya now sees what agents finished on their own**: kaya keeps an eye on every running agent's activity. When the target agent goes idle, kaya first checks queued tasks against on-screen evidence — a task the agent already completed is marked done and you're told, with no duplicate dispatch; one it's actively working on won't be nudged again.
- **Dispatch no longer stalls on authorization**: previously, dispatching to a tab without terminal-write approval only got you a refusal notice; now kaya pops the approval card along the way, and the task runs automatically once you tap Allow. Approvals granted in IM channels like Feishu are also recognized instantly on the desktop — no more "approved, yet dispatch won't go through."
- **Cleaner To Do List rows**: each task shows a status icon and its target agent in a single line, with actions (dispatch / retry / delete) tucked into a hover bar and failure reasons shown inline.

**🐛 Fixes**

- **Tapping Allow / Deny before the turn settled could later trigger an unexpected re-dispatch**: a stale "pending dispatch" was recorded in that window, and any later unrelated approval on the same tab could send the old task out again; registration now only happens while the card is still awaiting your decision.
- **Tapping Allow did nothing if the task vanished in the instant the card appeared**: in that rare race the approval landed on a task that no longer existed; kaya now re-confirms the task is still queued before dispatching.

## v2.3.8 (2026-09-22)

**✨ New Features**

- **Fast decision layer (on by default): instant answers for simple asks, a quieter watcher**: kaya adds a millisecond-scale decision layer — questions that need no tools (chit-chat, one-line Q&A) take a lightweight turn that's faster and cheaper; the coding-mode watcher now judges "is this worth interrupting you for?" before acting, so repeated idle reports and momentary "agent is running" states no longer spawn broadcast turns. Decision calls go to the same model you already selected, through the same scrubbing pipeline — no new data recipient. Turn it off any time in **Settings → Models**; it takes effect immediately.

**🚀 Improvements**

- **Quieter coding mode**: each watcher turn no longer spawns a fixed follow-up check turn at the end; a catch-up is only sent when you actually missed a notification. Overall notification volume drops noticeably.
- **Cheaper terminal reads**: when a terminal screen has no new output, re-reads return a one-line notice instead of re-sending the whole screen; and after a large output was truncated, you can now page back to earlier history with a larger read window.

## v2.3.7 (2026-09-21)

**✨ New Features**

- **Cross-agent ToDo List: you decide which agent takes a task, and when it runs**: a new To Do List area above the chat composer (collapsed by default, one click to expand). When the AI finds that a follow-up task needs a different agent, it writes the task — context included — into this list, where it survives new sessions and app restarts. Once the target agent goes idle, the AI asks "run it now?" and only your explicit yes dispatches it. When the run finishes, kaya verifies the result: a pass removes the entry; a failure stays with the reason attached, and you can retry with one click or ask the AI for help.
- **Your task list stays exactly as it was**: the familiar task list (checkboxes, folders, archiving) is unchanged and never mixes with the new To Do List.

**🚀 Improvements**

- **Project labels are now per pane**: two agents side by side in the same tab (e.g. two kimi code instances in the same directory) can now carry their own project labels instead of being forced to share one.

**🐛 Fixes**

- **After an agent exited, "Dispatch / Retry" on a task did nothing**: when a running agent was closed or the connection was re-established, tasks still pointed at the dead instance, so retrying looked like a no-op; the binding is now cleared as soon as the instance disappears, and you get a clear message when the target agent isn't running.
- **Confirming "run it now" in chat sometimes led nowhere**: if your confirmation happened to land while another chat turn was in flight, it was silently swallowed and you got asked again later; your confirmation is now remembered and the task is dispatched automatically as soon as the current turn ends.

## v2.3.6 (2026-09-20)

**✨ New Features**

- **Slack support: use kaya right from Slack**: the IM channel adds Slack (same experience as Feishu) — message the bot in Slack and your local kaya LLM streams its reply; when a terminal-write or config-edit approval is needed, an interactive Approve / Deny card arrives right in Slack, mutually recognized with the desktop grant cards. Setup: create an app at api.slack.com, enable Socket Mode, subscribe to message events, then paste the two tokens into Settings → IM Channel (tokens live only in the local system keychain), together with a member-ID allowlist.
- **Remote control (browser) now mirrors agent status**: the browser-based remote page shows each running agent's status line (model / thinking level / permission mode / context usage) and working directory, matching the desktop sidebar cards.

**🐛 Fixes**

- **Double-clicking a Slack grant card took effect twice**: rapidly double-clicking Approve / Deny sent duplicate confirmations and could trigger an extra follow-up turn; fixed — the same button on the same card now counts only once.
- **Mirror control named the wrong platform**: after switching the IM platform in Settings, the top-bar mirror button's tooltip could still name the previous platform; it now always follows the channel that's actually running.
- **Interrupted long replies ended incompletely in IM**: over-length content was rejected by the platform, leaving a bare "Interrupted" note; the message now closes with the (truncated) body plus the interruption marker.

## v2.3.5 (2026-09-20)

**✨ New Features**

- **Sidebar cards gain a status line: model / thinking level / permission mode / context usage**: see at a glance which model a running agent is on, its thinking level, its permission mode, and how much context it has left (kimi code shows all four; claude code shows the permission mode; codex shows the model; opencode is not supported yet). When space runs out only the key segments survive — hover for the full text.
- **The running-agents sidebar stays put in terminal-fullscreen**: previously, focusing the terminal (⛶ / ⌘⇧F) hid the sidebar along with the chat; now it stays docked at the window's left edge as a collapsible, drag-resizable panel.
- **Collapse the chat into a narrow rail**: a new ‹/› handle on the chat panel's right edge — click ‹ to collapse the chat and give the terminal the full width, click › to expand it back.
- **More "waiting for you" screens are recognized**: kimi code's question panel, codex's patch approval / permission requests / directory trust prompts, claude code's plan approval, and opencode's question panel and permission re-confirmation are all detected as "waiting for you", with the agent's option buttons listed right on the sidebar card — one click answers.

**🚀 Improvements**

- **Freely resizable sidebar**: both the inline sidebar next to the chat and the docked panel in terminal-fullscreen can be dragged wider or narrower via the right-edge handle.
- **Plainer permission-mode wording**: modes now read "Always ask / Ask when needed / Never ask" instead of opaque internal codenames.
- **Cleaner cards**: a tab's subtitle line only appears once the tab has been renamed, instead of always duplicating the title.

**🐛 Fixes**

- **kimi code 2.x status-bar parsing corrected**: the card's model and context usage didn't show, the permission mode was stuck on "default", and mode switches didn't update — all corrected against the real kimi code 2.x interface.
- **Task-list "mark done" checkbox was covered by the hover action bar**: in a narrow panel, hovering a task row made it impossible to tick the checkbox; fixed.
- **Top-bar model switcher button size**: it was one size larger than the neighboring icon buttons; now aligned.

## v2.3.4 (2026-09-19)

**✨ New Features**

- **Task queue: hand off work to a busy agent**: when one agent is busy and the next step needs another agent, enqueue the task — the target agent picks it up automatically when free, and kaya runs a verification round afterwards, notifying you on pass / fail. If your authorization is needed mid-task, it says so clearly. The task panel gains a "Queue" tab to view status, dispatch manually, retry, or delete; queue contents stay on your machine and survive restarts.
- **Archive tasks you're not doing yet**: archive moves a task into a collapsible section at the bottom of the panel, restorable anytime; archived tasks are hidden from the AI-visible list and don't take up numbering.
- **Label running agents with a project**: each terminal tab can carry an editable project label (click the project line on the sidebar card). When juggling multiple projects you can instantly tell which agent is working on what, and the @ picker can match by project name.
- **Click a running agent's sidebar card for a status report**: one click has kaya glance at the agent's terminal and briefly report what it's doing and whether it's waiting on you — no need to switch over and read line by line.

**🚀 Improvements**

- **Lower memory usage, faster reconnects**: terminal history (scrollback) now loads in streamed chunks instead of one giant read, so large logs no longer spike memory; several grow-only internal caches were also cleaned up, keeping memory steadier over long sessions.
- **Smarter @ picker filtering**: short queries (like `@u`) now prefer exact matches on name / project label instead of being drowned by agents sharing a directory; the loose broad matching remains as the fallback.

**🐛 Fixes**

- **Queue robustness at the edges**: when the target agent exits mid-task, verification keeps failing, or the app restarts, queue items now always resolve correctly (back to queued or marked failed) — no more stuck "in progress" items or duplicate dispatches.
- **Thorougher terminal startup recovery**: failure and stuck-replay paths now fully clean up internal state, so after reconnecting the pane renders normally instead of staying blank and silent.

## v2.3.3 (2026-09-18)

**✨ New Features**

- **Expandable event timeline in audit sessions**: click any session card on the Audit page to expand a per-event timeline — user inputs (with redaction counts), every assistant turn, and each tool call's decision and result (exit code + output), with errors highlighted and long outputs folded. "What did the AI actually do in that conversation" is now fully inspectable.
- **One-click retry on errors**: when a conversation fails due to network timeouts, provider errors, etc., the error banner gains a "Retry" button that resends your original message verbatim — no retyping.

**🐛 Fixes**

- **New tabs occasionally stuck blank**: a tab created via "+" or ⌘T could stay permanently blank (and silent) under special window-sizing timing; creation now retries automatically, and if it truly fails the tab shows a clear "disconnected" state with a reconnect entry.
- **Repeated auto-turns after resizing or splitting panes**: resizing the window or toggling pane splits was misread as "the other side started working", triggering duplicate automatic chat rounds; resize no longer disturbs busy/idle detection.
- **Old-tab auto-turns "reviving" after /new**: after starting a new conversation with `/new`, a waiting event from an old tab could incorrectly start a new round in the fresh session, or surface an `unknown tab_id` error; starting a new conversation now cleanly severs old bindings.
- **Fixed sensitive info in wrapped terminal echo possibly escaping redaction (privacy)**: extra-wide lines typed in the terminal (e.g. long card numbers) get re-drawn wrapped by the shell, which could split the digits into fragments the redaction rules no longer recognized. Impact: card-number-like info in terminal-read content under wrap-redraw scenarios. All users are advised to upgrade.
- **Unreliable Esc-to-stop**: Esc did nothing when focus wasn't in the input box; now Esc stops a streaming reply from anywhere and immediately shows a "stopped" confirmation. Also fixed Esc being misinterpreted as "stop reply" while the task panel, tab rename, IME candidate window, or top-bar menu had their own Esc semantics.
- **Pane-close confirmation wording**: closing a single pane no longer asks about closing the "Tab" — the message matches the actual behavior.
- **Invalid shell path silently disconnecting the terminal**: saving a nonexistent shell path in Settings used to leave the terminal Disconnected with no clear cause; the path is now validated before saving (exists / is a file / executable, relaxed on Windows), with a clear error and nothing persisted on failure.
- **Failure HTML flooding the settings UI**: when installing an agent failed behind a network block, the entire HTML error page rendered into the UI; error logs are now collapsed by default with over-long lines truncated.
- **Three issues with endpoint toggles**: turning off all three endpoints reported a generic error without saying "enable at least one"; disabling the OpenAI endpoint left the Agents panel unaware; disabling an endpoint discarded its saved values. Each is fixed — dedicated message, panel shows "endpoint disabled" and blocks binding, and disabling preserves values for a clean re-enable.
- **Settings semantics labeling**: apply-immediately controls (language, scrollback) are now labeled "Applies immediately", distinct from items that require clicking "Save".
- **Windows: shell shown in Settings didn't match the actual terminal**: on Windows, Settings could show a stale `/bin/zsh` while the terminal actually ran PowerShell; the displayed value is now reconciled to reality at launch, and the "follow system default" semantics are unaffected.
- **Audit evaluation results not visible**: after an evaluation finished, the report rendered below the session list with no visible change in the viewport; the view now auto-scrolls to the result.

**🚀 Improvements**

- **Built-in agent knowledge refresh**: official documentation snapshots for Claude Code / Kimi Code / OpenCode updated to latest (kimi-code's goal guide removed, mirroring the upstream docs).
- **Manual and copy alignment**: the model configuration entry is uniformly described as "Settings → Models"; API key storage wording unified to "your local system keychain, sent only to your configured model endpoint"; outdated configuration paths in the product manual ("General → LLM section") now describe the current flow (top-bar model switcher + Settings → Models).

## v2.3.2 (2026-09-16)

**✨ New Features**

- **Remote Access (cloud relay)**: keep work moving even away from your computer — view and operate this machine's kaya from a browser, with conversations, running-agent status, and approval prompts in real-time sync with the desktop. To enable: **Settings → Remote Access**, flip the switch (sign-in to your yunhou account and an active subscription required), then sign in on the yunhou website and open the "Remote Control" page. Privacy boundaries: off by default and explicitly opt-in; turning it off disconnects immediately; conversation content transits the server in memory only and is never persisted; **API keys never transit the relay** — this machine still connects to model providers directly; everything is TLS-encrypted. A customizable device name helps tell multiple machines apart, and the panel shows live connection state and recent activity.

**🐛 Fixes**

- **Two narrow-window fixes for the running-agents sidebar**: on small windows the sidebar becomes a floating drawer that could cover most of the chat area with no way out — clicking the backdrop outside the drawer now closes it; also, restarting the app on a narrow window no longer restores the sidebar as a screen-covering drawer — the expanded state only returns when the window is wide enough.

**🚀 Improvements**

- **Built-in agent knowledge refresh**: the official documentation snapshot bundled for Claude Code is updated to the latest version, making answers to "how do I use this agent" questions more accurate.

## v2.3.1 (2026-09-16)

**✨ New Features**

- **Mention a running agent with `@` right in chat**: type `@` in the chat input to list the currently running coding agents (keep typing to filter); picking one binds it as this session's forward target and completes the `@instance ID` into the input box, so you can see exactly who the message will go to — subsequent messages route to that agent without opening `/agents` first. It is the same binding channel as the sidebar cards and `/agents`.

**🐛 Fixes**

- **Fixed repeated "400 reasoning\_content" errors after many tool-call rounds**: with models that think by default (e.g. recent DeepSeek / Kimi / MiniMax versions), a long task could keep failing mid-run even with the thinking display toggle off. The reasoning content is now properly saved with the conversation and echoed back to the model; the interface still shows no thinking blocks while the toggle is off. Affected older sessions can recover by starting a new conversation.

**🚀 Improvements**

- **Built-in agent knowledge refresh**: the official documentation snapshots bundled for Claude Code / Kimi Code / OpenCode are updated to the latest versions, making answers to "how do I use this agent" questions more accurate.

## v2.3.0 (2026-09-15)

**✨ New Features**

- **Running-agents sidebar (coding mode only)**: a collapsible sidebar at the left edge of the chat panel gives you an overview of every running coding agent — each card shows the instance number (e.g. `kimi-code #2`, a global sequence), its current state (waiting for you / idle / running), and its tab and working directory. One click on a card switches to that tab, binds the agent as the forward target of the current chat, and focuses the input box — then just talk to it in chat. Cards waiting for a decision list the agent's option buttons inline, so you can answer with one click without switching tabs. The sidebar shares width with the chat at a fixed ratio, and on narrow windows it becomes a floating drawer instead of squeezing the chat.
- **Renamed agents are recognized too**: if you renamed a coding agent's CLI command (e.g. you use `myclaude` instead of `claude`), add those names in the "Command names" field on the agent's card (comma-separated). Install detection, foreground-process recognition, and the launch command all accept both the original and your custom names — no more false "not installed", and driving, `/agents` binding, and waiting/idle notifications all keep working for renamed agents.

**🚀 Improvements**

- **Built-in model "rejected by upstream (502)" self-healing with per-cause guidance**: when the upstream model service rejects a request, kaya first auto-compacts earlier conversation history and retries once (this only affects the AI's context — your visible chat history is unchanged), so most context-length cases recover without any manual action. If it still fails, the error card shows guidance for the specific cause (context too long / content review / insufficient quota) plus a one-click "New session" button (not shown for content-review and quota rejections, where a new session would not help).
- **Built-in agent knowledge refresh**: the official documentation snapshots bundled for Claude Code / Kimi Code / OpenCode are updated to the latest versions, making answers to "how do I use this agent" questions more accurate.

**🐛 Fixes**

- **Fixed tool-call markup occasionally leaking into the final reply**: in some cases the AI's final reply would contain a raw `<|DSML|…>` tool-call block together with a warning banner; these variant markers are now recognized and recovered properly instead of being displayed verbatim.

## v2.2.5 (2026-09-13)

**🐛 Fixes**

- **Blank chat on open and blank terminal after switching tabs now self-heal**: previously, opening a conversation or switching back to a terminal tab could leave the screen blank even though the content had been generated — scrolling once was the only way to make it appear. This version adds a self-healing fallback that repaints automatically, so no manual scrolling is needed.
- **Agents panel model list layout fixed**: fixed an issue where long hint text squeezed the model name, the "In use" badge, and the "Use" button into overlapping rows and clipped the bottom of the list; hints now sit neatly on their own line below the entry.
- **Locally built installer gains the install guide**: dmg packages built locally now show the same "drag to Applications" arrow as the official release builds.

## v2.2.4 (2026-09-13)

**✨ New Features**

- **The model library can declare a wire capability for OpenAI endpoints**: Codex officially supports only the Responses API, while Kimi Code / OpenCode speak Chat Completions. The OpenAI endpoint in Settings → Models now has a "Wire capability" choice (Chat Completions / Responses / Both); the Codex card only allows binding entries declared for Responses, disables the button with a pointer to the fix when the declaration is missing, and leaves existing bindings untouched (they just show a notice). The DeepSeek / Kimi Coding Plan / OpenAI presets prefill "Both".
- **Agents panel gains "Unbind"**: after binding, you can take an agent off that model at any time (reverting it to its own defaults) — only that one agent is affected; the entry and every other agent stay untouched; when nothing is bound, nothing changes.

**🚀 Improvements**

- **The Codex config panel now shows the launch command**: the model config kaya writes for Codex is only loaded under `codex --profile kaya` (deliberate isolation to coexist with the ChatGPT desktop app). The panel now shows that command and explains that a bare `codex` run will not see the configured model.
- **Updated DeepSeek preset**: model names are unified to the current official `deepseek-flash`, noting that DeepSeek officially supports the Responses API — fill `deepseek-flash` in the OpenAI endpoint to use DeepSeek with Codex directly.
- **More truthful agent install status**: when an agent is installed under a non-default Node version and is not actually usable from your terminal, the Agents panel says so and offers a one-click "Install into current environment" repair (it only runs when you click it).

**🐛 Fixes**

- **Root-cause fix for "Codex can't find the model I configured"**: fixed the probe of Codex's model catalog hanging on large output and the generation failure being silently ignored; failures now leave a diagnosable warning, and a stale catalog is regenerated on the next launch or library change.
- **"Use" no longer fails for Chat-Completions entries on Kimi Code / OpenCode**: previously the button was clickable but saving reported "no OpenAI endpoint"; binding and automatic sync now share one resolution with the same fallback rule.
- **Unbind status and hints corrected**: a partial unbind failure is no longer reported as a failed unbind; when an entry has no usable OpenAI endpoint the hint is actually actionable (enable and fill the endpoint first); other agent cards affected by an unbind refresh their status.

## v2.2.3 (2026-09-12)

**🐛 Fixes**

- **Terminal freezes and blank panes after switching focus are fully fixed**: previously, after pasting content into a coding agent such as OpenCode, input could stop responding, and switching focus could leave the pane blank with closing it as the only way out. With the root cause fixed, even if the same class of failure recurs, the affected terminal recovers automatically instead of taking down the pane and kaya's terminal-reading tools.

**🚀 Improvements**

- **Feishu / WeCom no longer spam tool-call messages**: the assistant used to send a separate message for every tool call (running a command, reading the terminal, …) — up to a couple of dozen per turn, all piling up below the reply. Now only **failed or blocked** calls produce a one-line notice (which tool did not succeed); successful calls stay silent.

## v2.2.2 (2026-09-11)

**🐛 Fixes**

- **Terminal freezing / permanently blank screen after focusing or switching panes (including OpenCode) is fully fixed**: v2.2.1 addressed only one of the causes; this release fixes all three that stacked up — a hidden terminal pane is no longer re-laid-out at a near-zero width, the clear now happens strictly in sync with the window-resize notification, and focus transitions no longer squeeze the terminal through intermediate widths. A running coding agent no longer freezes or blanks out from abnormal resize events.
- **Windows: `clear` / `cls` now actually clears the screen**: previously, when part of the Windows clear sequence was dropped to protect scrollback, the visible area kept stale content, so it looked like clear did nothing. The viewport is now always cleared while scrollback is still preserved.
- **Windows: `Ctrl+Alt+Arrow` no longer leaks escape sequences into the terminal**: that combination is the app's own pane-navigation shortcut; the arrow key's escape sequence used to be sent to the terminal as well.
- **Windows / git-bash: complete terminal initialisation**: terminals started under git-bash now get their own command history, idle/directory markers and working-directory tracking — the missing working-directory marker previously caused a "directory no longer exists, fell back to home" notice after every command.
- **Windows: more reliable detection of running coding agents**: fixed npm-installed CLIs (such as Kimi Code) not being detected on Windows, which broke "send to the agent", waiting-for-confirmation and `/agents` binding; also fixed agent update / uninstall being unavailable under the default Node install layout.
- **Windows: glob wildcards work again**: fixed `*` and similar patterns being silently treated as literals and not expanded when running commands under bash / git-bash.
- **Windows: saving keys no longer errors when many entries are configured** (the system credential store has a size limit), and a transient read failure can no longer wipe other stored keys by mistake.
- **Windows: corrected the global command-history location and command-extension parsing**: no more stray unused `.zsh_history`, and backslash paths no longer cause a wrong file extension to be derived (that parsing gates whether file redirection under `~/.kaya` is allowed — a security hardening, upgrading is recommended).

**🚀 Improvements**

- **Streamlined agent list**: removed the CC Switch and Reasonix entries, leaving the four collaborative CLIs Claude Code / Kimi Code / OpenCode / Codex; leftover documentation snapshots for the removed entries are cleaned up after upgrading.

## v2.2.1 (2026-09-11)

**✨ New Features**

- **Configure a model once — shared by Chat and coding agents**: Settings → Models _is_ the coding agents' provider library — one shared configuration, maintained in **Settings → Models** (adding, editing and deleting all live there; the Agents panel only binds an entry to an agent). One entry can carry three optional endpoints — Chat, Anthropic and OpenAI: Claude Code uses the Anthropic one, Kimi Code / OpenCode / Codex use the OpenAI one (falling back to the Chat one when it is empty), and the Chat side prefers the Chat one (falling back to the OpenAI one), so configuring once covers both sides. API keys now live in the system keychain and are **no longer stored in plain text in config files**; per-agent configurations from older versions are migrated and merged automatically on the first launch after upgrading.
- **Predictable proxy behaviour**: network requests kaya makes itself (chat, account, update checks, web search, channel connections) no longer follow the system proxy setting and **connect directly by default** — a machine-wide proxy will not change how the app behaves locally. Two exceptions: (1) Feishu / WeCom message traffic is sent by a third-party SDK and still reads the `HTTP_PROXY` / `HTTPS_PROXY` environment variables (this only happens when you launch kaya from a terminal that exports them — launching from Finder / the Start menu usually does not); (2) requests issued from inside the UI by the system WebView (such as the audit page's model evaluation) follow the WebView's own network settings.

**🚀 Improvements**

- **Models are configured in one place — Settings → Models — and the Agents panel only picks**: previously a model's endpoint, model name and API key could be edited both on the Settings page and in the Agents panel, and either copy could be written into an agent's config file, making it easy to end up with two conflicting setups. Models — including the Chat, Anthropic and OpenAI endpoints — are now **managed in one place**: add or edit an entry under Settings → Models, then bind it to an agent with Use in the Agents panel. The Agents panel no longer offers Add / Edit / Delete; when the list is empty it points you to the model list.
- **Model configuration can no longer be changed behind your back**: the AI assistant can still **read** an agent's config file while troubleshooting, but can **no longer change** the model name, endpoint address or key in it — such edits are refused, and it will point you to the model library instead.

**🐛 Fixes**

- **A coding agent running in focus mode no longer freezes the terminal or garbles the display**: fixed an issue where entering focus mode (the ⛶ button in the top-right, or ⌘⇧F / Ctrl+Shift+F) hid the terminal but did not actually remove the terminal panel from the layout — it was re-laid-out at a near-zero width, so a running coding agent (reproduced with OpenCode) repeatedly received tiny window-resize notifications, freezing itself and garbling the terminal, and in severe cases making the whole app unresponsive with tabs and panes unclickable. Focus mode now hides the terminal properly, so agents no longer receive abnormal resize events.

- **Deleting a model entry also deletes its API key** (the confirmation dialog says so); deleting the model currently in use falls back to the built-in model instead of leaving a dangling selection.

- **Saving an edited entry no longer deletes its API key by accident**: previously, hitting Save in the brief window between opening the editor and the key being read back could silently drop the stored key.

- More reliable upgrade migration: repeated launches after migration no longer produce duplicate entries, and legacy per-agent configuration is migrated correctly whether or not it carries a version number.

- **Editing or deleting a model entry now actually reaches the coding agent using it**: previously the config was written into an agent only at the moment you clicked Use. After that, changing the endpoint, rotating or clearing the API key, or deleting the entry in the model library left the agent untouched — a rotated key never took effect, and after deleting an entry the agent kept using the deleted model along with the deleted key (left behind in plain text inside the agent's config file). Now:

  - Editing an entry **automatically** updates every agent using it (Claude Code / Kimi Code / OpenCode / Codex) — no second click on Use required. Clearing the API key also removes the leftover key from the agent's config file.
  - Deleting an entry makes the agents bound to it drop kaya's configuration and fall back to their own defaults: they stop using the deleted model and key, and **no plaintext key is left behind** in their config files.
  - Agents that are not using the entry are not touched at all; your other fields and other provider configurations in those files are always preserved.
  - The **In use** badge in the panel now reflects the real binding (an entry whose values happen to match but is not actually in use no longer shows as in use). If an agent's config file was edited by another tool — the endpoint values changed, or the API key in it was changed or removed — the entry shows "Config file edited externally"; click Use to re-apply. (Previously these cases gave no warning and the Use button stayed disabled.)
  - kaya only takes over configuration it wrote itself: if you hand-wrote an agent's config file (even if its endpoint happens to match a library entry exactly), kaya will not treat it as in use, and deleting that entry later will not touch it — your own endpoint, model name and key stay exactly as you wrote them.
  - If one agent's config file is broken, only that agent is affected: the others are still updated, and the broken one is retried on your next action.
  - Leftover Codex traces from older versions are cleaned up too: early versions wrote directly into Codex's own config file; deleting an entry now strips those traces as well, leaving no settings pointing at something that no longer exists.
  - After upgrading, agents you had configured with Use are recognised and managed automatically; nothing to reconfigure (recognition requires the config file's current values to match exactly one library entry — if the values were changed, or the entry was deleted, the agent is not claimed).
  - **One exception**: when **kaya did not claim the config** (it carries no kaya-written marker — e.g. a Claude Code config kaya wrote before 2026-09-09 and never re-applied since, which predates that marker; its endpoint values match no library entry; or the entry was deleted before claiming could happen), the kaya settings and the plaintext key left behind in that agent's config file are **not cleaned up automatically** — click Use for that agent once to converge it, or restore the `.bak` backup next to the file by hand. (v2.2.0 and earlier kept no binding record, so after upgrading the first recognition relies on the config matching exactly one library entry.)
  - **Please note**: deleting an entry does not restore the config the agent had before kaya took it over (those values were overwritten when the config was applied and cannot be recovered) — restore the `.bak` backup written next to the file if you need the old configuration.

## v2.2.0 (2026-09-09)

**✨ New Features**

- **Centralized model provider management**: the Agents hub provider configuration is now a global registry — configure an API key once and it can serve Claude Code / Kimi Code / OpenCode / Codex at the same time; the preset list adds Kimi Coding Plan, MiniMax and more, and per-agent configurations from older versions are migrated and merged automatically.
- **Codex support**: the Agents panel can now detect, install, and launch OpenAI Codex, and configure its model provider centrally like any other agent. Note: Codex currently only works with responses-protocol endpoints; chat-only endpoints (e.g. DeepSeek direct) are not supported yet (an official Codex limitation).

**🐛 Fixes**

- **kaya's Codex configuration no longer interferes with the ChatGPT desktop app**: kaya previously wrote Codex configuration into the same file the ChatGPT desktop app uses, which could route the app's model requests to the wrong endpoint and break its official features. kaya's configuration now lives in an isolated profile, fully separate from the app; legacy configuration is migrated and cleaned up automatically.
- **Codex custom model usability**: fixed a "Model metadata not found" warning on every launch after configuring a custom model, and the configured model not appearing in the `/model` picker — custom models now show up in the picker normally.
- **Claude Code custom model trio of fixes**: fixed the unrecognized-model notice at launch, the `/model` Opus/Sonnet/Haiku tiers sending built-in model names to third-party endpoints (causing errors), and the configured model missing from the picker list.
- **More accurate agent detection**: fixed the Agents panel occasionally showing "unknown version" on some machines, and CLIs installed only under a non-default nvm version being misreported as "not installed".

## v2.1.17 (2026-09-08)

**🚀 Improvements**

- **A cleaner chat view**: banners from automatic turns (coding monitor, grant confirmations, scheduled tasks) now show a single factual line — e.g. "kimi-code is waiting for your confirmation: Run this command?" — instead of the full internal AI instruction verbatim. What gets sent to the AI is unchanged.
- **Friendlier error messages**: when a command is blocked by the safety policy, the notice now shows a readable explanation instead of an internal code (hover to see the original); timeout and tool-limit banners no longer append English technical details, keeping only actionable guidance; grant cards no longer display a process id that carries no actionable value.
- **More readable scheduled-task approval cards**: the summary shows the task's own text instead of a UUID; after you approve or deny, the card keeps the summary as of your decision — even if the task was renamed or deleted while the card was pending, it never degrades to a bare id.
- Chinese UI polish: the "system" role label is localized, and three notice banners restored proper fullwidth punctuation.

**🐛 Fixes**

- **AI couldn't find commands during Feishu/WeCom remote sessions**: in the installed app, remote turns ran commands with a minimal environment, so tools like brew / nvm / cargo reported "command not found"; remote turns now use exactly the same environment as desktop chats. On Windows, also fixed: CLIs installed to a custom npm directory were invisible, and launching from a customized terminal could drop PATH entries.
- **Automatic turns yield better to remote replies**: coding-mode automatic turns now check for an in-progress remote reply before firing, and quietly retract if rejected — no more false "asking AI to continue…" banners; rare timing edges where conversation state could be reset or a reply could go unrecorded are fixed as well.

## v2.1.16 (2026-09-06)

**🚀 Improvements**

- **Friendlier guidance after your subscription ends**: when your subscription or trial expires, the built-in model prompt no longer only pushes renewal — it now makes clear you can keep chatting by switching to your own model (BYOK), with a new "Use my own model" button: one click switches to an already-configured profile, or opens the model settings page if you don't have one yet. The same guidance applies to entries like task extraction; sign-in and subscription errors no longer show internal technical fields.
- **More transparent channel status**: the Feishu/WeCom channel panel now shows "last message received N minutes ago / never" and a notice when messages from non-allowlisted senders are blocked; titles of notifications from remote sessions no longer show garbled placeholders.

**🐛 Fixes**

- **Crash on launch (critical)**: the initial v2.1.16 build could not be opened — it crashed instantly at startup for all users. This re-release fixes it; if you downloaded v2.1.16, please reinstall.
- **A channel no longer keeps running after the subscription expires, is cancelled, or you sign out while it's active**: the channel now stops automatically, the remote-coding master switch turns off with a notice, and it reconnects automatically after you renew. Please upgrade soon.
- **Several silent "message sent to Feishu, no response" scenarios**: rich-text messages are now handled (text is extracted and answered); unsupported message types (files, stickers, merged forwards) get a clear "not supported yet" reply instead of silence; messages that only @-mention the bot with no content get a guidance reply; truncation of very long command output is now actually surfaced.
- **Remote-coding cards occasionally stuck or duplicated**: fixed "thinking…" cards that could freeze forever or appear twice when messages arrived in quick succession; interrupted turns now correctly show "interrupted"; grant cards only take effect once on double-clicks or duplicate deliveries, and cards issued more than an hour ago are rejected with a notice.
- **Desktop auto-turns could interrupt a Feishu reply in progress**: automatic desktop turns in Coding mode and remote message turns now yield to each other properly instead of interrupting.
- **Channels occasionally couldn't be stopped or showed no status after start**: fixed orphaned connections from rapid repeated starts; manually stopping a channel now also turns off its "enabled" switch so it won't silently auto-start on the next launch; a channel that dies unexpectedly now reconnects with backoff.
- **AI "thinking drafts" leaking into replies**: some models mixed their reasoning directly into the reply text shown in chat and Feishu; it is now stripped before delivery (shown under the "thinking" section when thinking mode is on), and already-polluted history is cleaned automatically.
- **macOS fullscreen rendering only the top half**: on some macOS versions (e.g. macOS Tahoe 26.6 Beta) with an external display, restoring a fullscreen window at launch could render only the top half of the page (an upstream WebKit issue; leaving and re-entering fullscreen recovered). The window geometry is now verified and corrected automatically after fullscreen transitions; if you still hit it, diagnostic details are logged — feedback is welcome.
- **Coding / mirror toggles could disagree with the actual state**: failed toggle operations now roll back the UI and show an error, instead of "looks on, actually off".

## v2.1.15 (2026-09-05)

**🐛 Fixes**

- **Feishu "can receive but can't reply"**: after a Mac wakes from sleep, replies sent from kaya to Feishu could silently fail for a long time (up to about two hours before recovering on its own), while incoming Feishu messages kept arriving. Expired tokens are now force-refreshed with an automatic resend, and sends that fail right after wake (network not ready yet) get one automatic retry.
- **Terminal no longer stays scrolled up when you type**: previously, after long output like `ps aux`, scrolling up and then typing left the view stuck in the scrollback; now any input scrolls back to the bottom first, matching iTerm behavior.
- **Settings tab titles no longer wrap vertically**: with more settings tabs, titles like "General" were squeezed into one character per line; tabs now stay on a single line and scroll horizontally when they overflow.
- **Terminal column mismatch after moving the window across displays**: dragging the window between a Retina display and a standard-resolution external monitor (or changing Windows display scaling) could leave a right-edge gap or a one-column mismatch; the terminal now re-fits automatically.
- **Tab rename input could exit editing unexpectedly**: on a busy system, the inline rename input could lose focus to the background terminal right after opening (losing the rename); switching tabs no longer steals focus from input fields.

## v2.1.14 (2026-09-05)

**✨ New Features**

- **More BYOK providers**: the custom-model provider presets grow from 5 to 11, adding Kimi (Moonshot), GLM (Zhipu), MiniMax, Qwen (DashScope), SiliconFlow, and OpenCode Zen. Default models follow each vendor's current lineup, and thinking-capable models automatically get the right reasoning parameters.
- **Usage statistics**: helps us understand overall product usage (active days, usage duration, platform and version distribution). On by default when signed in and can be turned off anytime in "Settings → Account". It never includes conversation content, file content, or personal identity information, and nothing is reported while signed out.

**🚀 Improvements**

- When local storage fills up, instead of an easy-to-miss transient toast, a prominent banner now appears in the chat with actionable guidance (click "+" at the top to start a new session and free space; your history remains fully preserved in the local audit log).

## v2.1.13 (2026-09-03)

**✨ New Features**

- **Manage scheduled tasks from chat**: ask the AI in a conversation to list, add, remove, or enable/disable your scheduled tasks (e.g., "add a task that summarizes terminal output every morning at 9"). Any change shows a one-time authorization card and only takes effect after your approval.
- **Run-count limit for scheduled tasks**: when creating a task you can now set a maximum number of runs (e.g., "every minute, 10 times"); the task disables itself once the limit is reached while keeping its history. The settings form gains an optional run-limit input, and the list shows run progress.

## v2.1.12 (2026-09-02)

**✨ New Features**

- **Scheduled tasks**: create automation tasks in "Settings → Schedules" to let kaya automatically start an AI chat turn on your behalf every N minutes or at a fixed time each day (e.g., check build status every ten minutes, summarize terminal output every morning at 9). Tasks only fire while the app is running.
- **Tab renaming**: double-click a tab title to rename it, making parallel sessions easier to tell apart; duplicate names are rejected with a prompt, and names persist across restarts.

**🐛 Fixes**

- Fixed an issue where closing the focused terminal pane left keyboard input "dead" (focus lost to the page background); focus now moves to a neighboring pane automatically.

## v2.1.11 (2026-09-01)

**🌐 Much stronger resilience on slow or unstable networks**

- When the network slows down or drops, AI chat handshakes now retry automatically and honor server rate-limit backoff hints, noticeably improving chat success rates on weak networks.
- Failed QR-code logins on weak networks now retry with backoff instead of forcing a fresh scan; while offline, a "network issue, retrying in background" state is shown with a manual retry button.
- Feishu/WeCom channels: outbound sends now have timeout protection, dropped connections reconnect with jittered backoff, and abnormal exits are logged; long silent hangs no longer occur.
- Agent install/update/uninstall now has timeout protection — a stuck npm no longer leaves the card on "installing" forever; web search retries automatically with backoff when rate-limited.
- Audit evaluations now have timeout protection and no longer stay on "evaluating" forever on half-open connections.

**🐛 Fixes**

- Fixed an issue where the Feishu/WeCom mirror toggle appeared on after a restart but messages were never mirrored; the top-bar mirror icon is now tri-state (grayed out with guidance when the channel is not running / not bound, a normal toggle once bound).
- Fixed an issue where "task completed" was falsely reported while a coding agent's background tasks were still running; completion notices now include a summary of the task's last real output and use the wording "seems to have finished."
- The dmg installer window now has a drag-and-drop guide background image.

## v2.1.10 (2026-08-31)

**🔒 Security Update**

- Tightened the policy boundary for AI-executed shell commands, fixing several policy-bypass risks (covering file reads, command execution, download validation, and other paths).
- Hardened validation for "sending input to a coding agent": input carrying terminal control sequences or invisible Unicode characters is rejected, and the target state is re-verified before the submit action executes.
- Tightened agent configuration write boundaries: configuration items that could let a coding agent execute arbitrary commands or escalate privileges are now forbidden to modify.
- Output scrubbing now recognizes 8 additional common secret formats (JWT, Slack, npm, PyPI, SendGrid, DigitalOcean, Shopify, Twilio, and more).
- **All users are advised to upgrade as soon as possible.**

**🚀 Performance & Resources**

- Terminal output transport switched to an efficient encoding; the UI is noticeably more responsive with large outputs.
- Background monitoring automatically throttles down when idle: with no coding agent running, CPU usage drops significantly.
- Session history now has a memory cap, greatly reducing memory usage over long sessions.
- Reduced terminal log write overhead; audit logs are now automatically retained for 90 days instead of accumulating indefinitely.

**🐛 Fixes**

- Fixed an issue where certain internal errors could leave chat permanently stuck on "Generating"; errors now surface properly and the chat recovers.
- Fixed an issue where the Feishu/WeCom channels could become unresponsive for long periods under abnormal conditions.
- Fixed an issue where slow-loading shell configurations (nvm/conda, etc.) could occasionally cause terminal environment variables to be lost.
- Fixed an issue where chat history was silently lost when local storage filled up; a clear prompt is now shown.
- Output from the agent installation process is now scrubbed before being displayed, avoiding brief exposure of credentials that install scripts may emit.

## v2.1.9 (2026-08-30)

**🐛 Fixes**

- Fixed an issue on Windows where the split-pane shortcuts (Ctrl+Shift+D / Ctrl+Shift+S) could leave terminal panes "frozen" (unable to type or close).

## v2.1.8 (2026-08-29)

**✨ New Features**

- **Model configuration is now a first-class citizen**: save multiple model profiles, switch between the built-in model and your own models with one click from the new model switcher in the top bar; each custom model's API key is stored independently, and legacy configurations are migrated automatically.

**🐛 Fixes**

- Fixed an issue where instructions sent to kimi-code occasionally "typed but never executed."
- Fixed an issue where the Feishu/WeCom channels would not automatically reconnect after login state was restored, leaving them offline long-term.
- Fixed an issue where coding mode repeatedly flooded the screen with error toasts when not logged in / not subscribed.
- Fixed an issue where, in some cases, a valid login session existed in the keychain but the UI showed "not logged in" for a long time.

## v2.1.7 (2026-08-28)

**✨ New Features**

- The Agents panel now supports **uninstalling** installed coding agents (with confirmation, limited to kaya-managed installations).
- IM channel enhancement: when a coding agent is waiting for confirmation or has finished, the event is now injected into the IM session so the AI can perceive and handle it — no longer just a one-way notification.

**🐛 Fixes (from real Windows device feedback)**

- Fixed garbled text when reading UTF-8 files on Chinese Windows (PowerShell 5.1).
- Fixed an issue where, with a custom npm global directory, an agent showed "installation complete" but could not be detected or found in the terminal; when post-install detection fails, clear remediation guidance is now shown.
- Improved coding mode's perception of coding agents: newly launched agents are detected promptly, status reporting is more accurate, and occasional false reports are reduced.

## v2.1.6 (2026-08-27)

**✨ New Features — First Windows Desktop Release (x64)**

- kaya officially supports Windows: PowerShell 7 preferred, Windows PowerShell 5.1 as fallback, and git-bash users automatically keep their original shell.
- Terminal experience aligned with macOS: independent session per tab, history, and user profile loaded for the command-line environment.
- Shortcuts mapped per platform: Windows uses the Ctrl+Shift family (corresponding to the ⌘ family on macOS).
- Windows adaptation for agent detection/installation: automatically discovers npm install paths and retries via a China mirror when direct npm access fails.
- Coding mode fully functional on Windows: foreground coding agent detection, authorization cards, and interaction-waiting notifications are all restored.
- Windows installers released (msi / nsis); in-app update guidance prefers nsis.

**🐛 Fixes**

- Fixed GBK garbling of command output on Chinese Windows.
- Fixed console windows briefly flashing when executing commands on Windows.
- Fixed an issue where Node/npm installed while kaya was running required a restart to be recognized.
- Feishu/WeCom channel connection errors are now visible and inspectable on Windows (Settings → AppLog); the channel panel shows the most recent connection error in real time.
- Built-in model now automatically retries on network jitter and renews expired logins, with clearer error messages.
- claude-code gained a DeepSeek (Anthropic-compatible) provider preset for one-click configuration.

**⚠️ Known Limitations (first Windows release)**

- Certain PowerShell syntax may be incorrectly rejected by the safety policy (conservative rejection; normal commands are unaffected).
- Background "grandchild processes" may fail to be cleaned up when a command finishes in a few scenarios.

## v2.1.5 (2026-08-26)

**✨ New Features**

- **Two-tier feature gating**: sign in to use bring-your-own-key (BYOK) chat for free; a paid subscription unlocks the built-in model, coding mode, and IM channels. Friendlier in-app guidance when not logged in / not subscribed.

**🐛 Fixes**

- Fixed an issue where, after approving authorization via a Feishu card, an extra manual message was occasionally needed to resume execution.
- Fixed an issue where the channel start button / upgrade prompts did not refresh automatically after payment.

## v2.1.4 (2026-08-25)

**🐛 Fixes**

- macOS keychain password prompts at startup reduced from 3–5 times to at most 2 (key storage consolidated; rolling back to an older version requires re-entering API keys in Settings).
- Fixed an issue where Enter was occasionally swallowed when sending instructions to kimi-code, leaving the instruction stuck in the input box.

## v2.1.3 (2026-08-25)

**🐛 Fixes**

- Fixed intermittent "tool_call_id is not found" 400 errors during long sessions when using OpenAI-compatible models such as Moonshot/Kimi.

## v2.1.2 (2026-08-24)

**🚀 Improvements**

- Greatly improved model context cache hit rate: faster responses and lower token consumption in continuous conversations.

**🐛 Fixes**

- Fixed an issue where conversation trimming could, in some cases, cause Kimi to report "tool_call_id is not found."
- Fixed an issue where system instructions could be lost during multi-step tool calls, causing abnormal agent behavior.
- Fixed an issue where a manually started channel would not reconnect after an app restart; the mirror toggle now stays consistent with the settings page state.

## v2.1.1 (2026-08-23)

**🐛 Fixes**

- Fixed an issue where the Feishu/WeCom channels occasionally showed "Running" but could not actually receive messages (added a connection watchdog with timeouts and automatic reconnection).
- Fixed an issue where, with mirroring enabled, an ongoing desktop conversation could be mistakenly interrupted or messages could leak across sessions.
- Fixed an issue where stopping/restarting a channel could occasionally abort an in-progress desktop reply.
- Fixed an issue where the channel allowlist input box could not accept line breaks properly.
- Fixed an issue where channel component logs could be written to the system log; sensitive information no longer leaks.

## v2.1.0 (2026-08-23)

**✨ New Features — IM Channels**

- **Feishu integration**: connect a Feishu bot to your local kaya — chat and get work done with AI directly in Feishu, no public IP required.
- **WeCom integration** (entry temporarily greyed out as "Coming soon" in this version).
- **Desktop ↔ IM two-way mirroring**: desktop conversations are pushed to IM in real time, and replies in IM appear on the desktop in real time.
- **Authorization cards**: actions requiring approval (such as sending input to a coding agent) are pushed to IM as cards — approve/reject with one click, and execution resumes automatically after approval.
- **Proactive agent status notifications**: when a coding agent is waiting for confirmation / finished / online / exited, a notification is pushed to IM.
- **Feishu scan-to-connect**: create a Feishu app with one click and complete configuration automatically by scanning a QR code.
- New "IM Channels" panel in Settings; the access allowlist rejects strangers by default, and credentials are stored only in the system keychain.

## v2.0.5 (2026-08-21)

**✨ New Features**

- **Install/update coding agents directly in chat**: tell the AI "install claude-code for me" and it's done — sharing the same engine as the Agents panel; a global notification appears when installation/update completes.

## v2.0.4 (2026-08-21)

**✨ New Features**

- Coding mode monitoring loop closed: when a coding agent "starts working / finishes / comes online / exits," it's detected and reported promptly — no more missed reports over long runs.
- Confirmation menus of opencode / reasonix are now recognized.

**🐛 Fixes**

- Fixed an issue where long instructions were occasionally sent to a coding agent twice.
- Fixed an issue where installing/updating an agent picked the wrong Node environment and failed.
- Fixed an issue where agent events in split panes were silently dropped.

## v2.0.3 (2026-08-21)

**🐛 Fixes**

- Fixed an issue where updating an agent used the wrong npm, installed to the wrong location, or failed on permissions (the npm matching that agent is now selected automatically).
- Fixed an issue where the wrong Node version was picked up when using nvm.
- When installation/update fails, the tail of the output log is preserved and the card offers a fallback "Install/Update in Terminal" entry; failure reasons are no longer just an exit code.

## v2.0.2 (2026-08-20)

**✨ New Features**

- Task list supports group filtering (All / Ungrouped / each folder).
- Tasks are sorted by "In Progress > Not Started > Completed."
- Task titles wrap to two lines; action buttons now appear on hover (no longer squeezing titles, and accessible to keyboard/screen-reader users).

## v2.0.1 (2026-08-20)

**✨ New Features**

- kaya can directly answer usage questions about coding agents like claude-code / kimi-code / opencode / reasonix (official docs built in, queried locally).
- Coding mode understands each agent's native commands: say "set the mode to auto" and it's automatically translated into the corresponding command and executed.
- Task list gained an "In Progress" status; tasks can be organized into folders.

**🐛 Fixes**

- Fixed an issue where ⌘W / closing a tab occasionally popped up two confirmation dialogs.
- Fixed an issue where brand-new users started with no tabs at all.
- Fixed an issue where the upgrade/renewal entry occasionally returned 404.

## v2.0.0 (2026-08-19)

**✨ New Features — Product Positioning Upgraded to "Mission Control for AI Coding Agents"**

- **Task Panel**: tasks recognized in chat flow into the top task panel — view, edit, and organize them.
- **Agents Management Center**: one-stop install, update, and configuration of coding agents (Claude Code / Kimi Code / OpenCode, etc.), with silent background installation and one-click model configuration writes.
- **Coding Mode**: monitors coding agents running in the terminal and proactively reports when they're waiting for confirmation / finished; once authorized, it can send instructions to agents on your behalf.
- **Interaction confirmations straight to chat**: when an agent pops up an approval menu, you're notified in chat — just type the option number to respond, no need to switch back to the terminal.
- **Chat slash commands**: `/agents` binds a session to a running agent, `/new` starts a new conversation, `/sessions` shows history, `/help` shows help; unknown commands are automatically forwarded to the bound agent.
- **Consolidated menus and an icon-based top bar** for a cleaner interface.
- New international edition (GitHub login + PayPal payment).

**🐛 Fixes**

- Fixed an issue where chat output didn't auto-follow scrolling, leaving the viewport stuck at the top in long sessions.
- Fixed an issue where coding agents with inline terminal UIs (kimi/claude) were previously completely invisible to monitoring.
- Multiple driving-chain experience fixes: automatic continuation after authorization, monitoring turns no longer repaint the whole screen, and "typed but never submitted" is eliminated.

## v1.2.6 (2026-08-13)

**🐛 Fixes**

- Fixed an issue where DeepSeek occasionally output tool-call markers as plain text without executing the tools.
- Fixed an issue where improper key-release order caused the terminal to keep auto-typing.

## v1.2.5 (2026-08-13)

**✨ New Features**

- **Terminal multi-pane splitting** (iTerm2 style): `⌘D` splits left/right, `⌘⇧D` splits top/bottom, `⌘⌥arrow keys` move focus, drag to adjust ratios — each pane is an independent session.
- **Focus mode `⌘⇧F`**: temporarily zoom the current pane; exiting restores the split layout.
- Vim long-press movement fixed: holding `hjkl` moves continuously, matching iTerm2 behavior.

**🐛 Fixes**

- Fixed an issue where clicking external links had no effect.
- Fixed garbled lines appearing when scrolling after splitting panes.

---

## Earlier Versions

> Early records for v1.0.0 – v1.2.4 are yet to be filled in.
