# Release Notes

> User-facing release notes, sorted by version in descending order. Each version's new features, improvements, and fixes are described for end users, without internal implementation details.
> Maintenance convention: update this file on every release (the website consumes it as the release notes); security fixes describe only the impact and upgrade advice, never the exploitation details. Internal engineering records live in `CHANGELOG.md`.
> This is the English mirror of `RELEASE_NOTES.md` — keep both in sync.

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
