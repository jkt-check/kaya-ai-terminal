# kaya · 为 AI 开发者打造的 AI 终端

> **为 AI 开发者打造的 AI 终端**

> [English README](README.en.md)

kaya 是运行在**你自己的 Mac / Windows PC** 上、**为 AI 开发者打造的 AI 终端**。它以完整终端为基础，把模型管理、编程 Agent 管理、安全网关和 AI 大脑放进同一个开发工作区。

## 核心能力与完整终端

| 核心能力 | 当前支持范围 |
|---|---|
| **模型管理与多供应商** | 统一模型库按「供应商 → 模型」管理，200+ 家供应商目录预填端点与模型；支持自备 API Key、供应商 OAuth、自定义兼容端点和 Ollama 本地模型，凭据与协议适用范围见用户手册 |
| **编程 Agent 管理** | 安装、配置、更新 Claude Code / Codex / OpenCode / Kimi Code，绑定模型，在独立标签页中查看运行状态与终端输出 |
| **安全网关** | 可选、默认关闭的本机模型网关；仅对明确接入的编程 Agent 模型请求按策略检查、拦截或脱敏，支持的替换在响应中尝试还原，保护记录可查看 |
| **AI 大脑** | 结合终端上下文解释命令、错误与 Agent 方案，读取进展、整理结果和下一步；经 coding 模式及逐标签页授权后向受支持的 Agent 输入指令、交接任务与上下文 |

**完整终端是产品基础**：macOS 上使用 zsh，Windows 上使用 PowerShell，保留你的 shell 配置、alias、脚本和命令行工具，支持独立 PTY、多标签页与多窗格。用户自己的终端操作与 AI 的受限工具执行是两条独立路径。

**核心能力不是全部能力**：聊天里敲 `@` 或 `/agents` 可以绑定运行中的 Agent，多个 Agent 真正并行；跨 Agent ToDoList、跨机协同、飞书 / Slack（企业微信即将支持）、浏览器远程访问、记忆、定时任务、联网搜索、图片输入、会话历史、用量统计、隐私审计、配置优化与自动更新等现有能力同样保留。核心分类不改变其支持状态、订阅要求或授权边界；Privacy First、快速、简洁高效仍是产品原则。详见[用户手册](docs/PRODUCT.md)。

## 隐私与安全

**隐私第一**：默认 LLM 不能任意读取本机文件正文；只有你主动放入 `~/.kaya/` 可信目录、且未被敏感控制面规则拒读的内容，才允许受限工具读取。聊天侧由用户输入 guard、policy gate、safety gate、execute 隔离和 output scrub 共同保护；AI 工具在独立子进程执行，不直接向你的普通 shell 提示符输入命令。

模型网关与聊天是独立路径，不是全系统透明代理，也不会保护未接入 Agent 的直连流量。API Key / OAuth token 存在系统钥匙串，仅用于相应服务的鉴权；网关托管配置只写本机回环地址、占位 token 与模型别名。网关隐私明细可能含命中原文与替换令牌，仅存本机、默认遮罩，不自动进入模型上下文或诊断导出。

会话、记忆、审计与设置保存在本机，不自动云同步；选用云端模型时会发送对应内容，显式启用的搜索、IM、远程访问与跨机协同也有各自的传输路径。登录后的使用情况心跳可随时关闭；**云端 Token 用量上传默认关闭，需明确同意**，不上传对话或工具正文。Ollama 本地推理不外发推理与对话 / 工具内容，但不自动关闭上述独立通道。完整边界见[隐私设计文档](docs/PRIVACY.md)。

## 本仓库是什么

这是 kaya 的**公开文档仓库**，只包含对外发布的内容，**不含源代码**：

| 内容 | 中文 | English |
|---|---|---|
| 用户手册 | [docs/PRODUCT.md](docs/PRODUCT.md) | [docs/PRODUCT.en.md](docs/PRODUCT.en.md) |
| 隐私设计文档 | [docs/PRIVACY.md](docs/PRIVACY.md) | [docs/PRIVACY.en.md](docs/PRIVACY.en.md) |
| 常见问题 FAQ | [FAQ.md](FAQ.md) | [FAQ.en.md](FAQ.en.md) |
| 发布说明 | [RELEASE_NOTES.md](RELEASE_NOTES.md) | [RELEASE_NOTES.en.md](RELEASE_NOTES.en.md) |

## 下载安装

安装包统一在 **[kaya 官网下载页](https://www.yunhouai.com/download)** 发布：

- **macOS**：下载 `kaya-v<版本>-macos-universal.dmg`（Apple Silicon 与 Intel 通用），拖入 `/Applications` 即可。
- **Windows (x64)**：下载 `kaya-v<版本>-windows-x64.exe`（nsis，推荐）或 `.msi`，双击按向导安装。

详细步骤见 [用户手册 · 安装 kaya](docs/PRODUCT.md#安装-kaya) 与各版本 [发布说明](RELEASE_NOTES.md)。

## 反馈与问题

- 使用问题先查 [FAQ](FAQ.md) 和 [用户手册](docs/PRODUCT.md) 的「常见问题 / 故障排查」章节。
- 仍有问题请在本仓库提 [Issue](../../issues)。

## 许可

本仓库文档以 [CC BY 4.0](LICENSE) 协议发布。kaya 软件本身的许可以应用内「关于」对话框为准。
