# kaya — AI 编程 Agent 总控台

> **Mission Control for AI Coding Agents · Privacy First · Fast · Simple & Effective**

kaya 是一个跑在**你自己的 Mac / Windows PC** 上的 AI 编程 Agent 总控台：一个界面边聊天边 coding——LLM 可以调 shell 命令、读终端输出，coding 模式授权后可直接向 tab 里的编程 agent（Claude Code / Kimi Code / OpenCode / Reasonix 等）发送输入；`/agents` 把对话绑定到运行中的 coding agent，每 tab 独立 PTY，多 agent 真正并行。

**隐私第一**：默认 LLM 看不到你的任何文件内容——这是结构性的，不是配置项。所有用户输入与工具输出在出门前都要过 5 道关卡（用户输入 guard → policy gate → safety gate → execute 隔离 → output scrub），全部状态留在本机，无云同步、无遥测。

## 本仓库是什么

这是 kaya 的**公开文档仓库**，只包含对外发布的内容，**不含源代码**：

| 内容 | 位置 |
|---|---|
| 用户手册（中文 + English） | [docs/PRODUCT.md](docs/PRODUCT.md) |
| 隐私设计文档（中文 / English） | [docs/PRIVACY.md](docs/PRIVACY.md) · [docs/PRIVACY.en.md](docs/PRIVACY.en.md) |
| 常见问题 FAQ（中文 / English） | [FAQ.md](FAQ.md) · [FAQ.en.md](FAQ.en.md) |
| 发布说明（中文 / English） | [RELEASE_NOTES.md](RELEASE_NOTES.md) · [RELEASE_NOTES.en.md](RELEASE_NOTES.en.md) |

## 下载安装

- **macOS**：下载 `kaya-<版本>-macos-universal.dmg`（Apple Silicon 与 Intel 通用），拖入 `/Applications` 即可。
- **Windows (x64)**：下载 `kaya-<版本>-windows-x64.exe`（nsis，推荐）或 `.msi`。

具体步骤见 [用户手册 · 安装 kaya](docs/PRODUCT.md) 与各版本 [发布说明](RELEASE_NOTES.md)。

## 反馈与问题

- 使用问题先查 [FAQ](FAQ.md) 和 [用户手册](docs/PRODUCT.md) 的「常见问题 / 故障排查」章节。
- 仍有问题请在本仓库提 [Issue](../../issues)。

## 许可

本仓库文档以 [CC BY 4.0](LICENSE) 协议发布。kaya 软件本身的许可以应用内「关于」对话框为准。

---

## English

**kaya** is a Mission Control for AI coding agents that runs on **your own Mac / Windows PC**: chat and code in one window — the LLM can run shell commands and read terminal output, and with coding-mode grants it can type into coding agents (Claude Code / Kimi Code / OpenCode / Reasonix) running in tabs. Bind a conversation to a running agent with `/agents`; each tab gets its own PTY, so multiple agents run truly in parallel.

**Privacy first**: by default the LLM cannot see any of your file contents — this is structural, not a setting. Every user input and tool output passes through 5 gates (input guard → policy gate → safety gate → isolated execution → output scrub) before leaving your machine. All state stays local; no cloud sync, no telemetry.

This repository contains **public documentation only — no source code**:

- User guide (中文 + English): [docs/PRODUCT.md](docs/PRODUCT.md)
- Privacy design (中文 / English): [docs/PRIVACY.md](docs/PRIVACY.md) · [docs/PRIVACY.en.md](docs/PRIVACY.en.md)
- FAQ (中文 / English): [FAQ.md](FAQ.md) · [FAQ.en.md](FAQ.en.md)
- Release notes (中文 / English): [RELEASE_NOTES.md](RELEASE_NOTES.md) · [RELEASE_NOTES.en.md](RELEASE_NOTES.en.md)

Docs are licensed under [CC BY 4.0](LICENSE).
