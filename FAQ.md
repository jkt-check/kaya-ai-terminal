# kaya 常见问题（FAQ）

> 更完整的使用说明见 [用户手册](docs/PRODUCT.md)；隐私设计的实现细节见 [docs/PRIVACY.md](docs/PRIVACY.md)；各版本变化见 [发布说明](RELEASE_NOTES.md)。
>
> [English version](FAQ.en.md)

## 目录

- [安装与平台](#安装与平台)
- [隐私与安全](#隐私与安全)
- [模型与配置](#模型与配置)
- [终端与标签页](#终端与标签页)
- [编程 agent 协同（coding 模式）](#编程-agent-协同coding-模式)
- [IM 渠道（飞书 / Slack / 企业微信）](#im-渠道飞书--slack--企业微信)
- [远程访问（浏览器）](#远程访问浏览器)
- [故障排查](#故障排查)

---

## 安装与平台

**支持哪些平台？**

macOS（Apple Silicon 与 Intel 通用，dmg 安装包）和 Windows x64（msi / nsis 安装包，推荐 nsis）。Linux 暂不支持。安装包统一从 [kaya 官网下载页](https://www.yunhouai.com/download) 下载。

**macOS 提示"无法验证开发者"怎么办？**

在 Finder 里右键 `kaya.app` →「打开」，或在 系统设置 → 隐私与安全性 中允许。不要关闭 Gatekeeper。

**为什么 macOS 弹"是否允许 kaya 访问 Desktop / Downloads / Documents"？**

这是**正常的 macOS 行为**，不是 kaya 在偷偷要权限。当 AI 调用 shell 命令（如 `ls ~/Desktop`）时，kaya 会 spawn 一个独立子进程，macOS 的 TCC 机制会对首次访问受保护目录的 app 弹确认框——iTerm2、WezTerm、VS Code Terminal 等所有 spawn 系统 shell 的应用都一样。

- 点「允许」一次，该目录之后不再弹（每个新目录仍会问一次，这是 macOS 设计，无法在 app 层消除）。
- 如果你经常让 AI 跑 shell，可以一次性给 kaya 完全磁盘访问（系统设置 → 隐私与安全性 → 完全磁盘访问 → 添加 kaya.app）。kaya 的 5 道隐私关卡仍然保证 LLM 默认看不到你的文件内容；FDA 只是让 kaya 的终端子进程能正常工作。
- macOS Sequoia 起系统每月会重新询问一次屏幕录制权限，这是系统行为，所有桌面 app 都会遇到。

**Windows 上用哪个 shell？**

默认 PowerShell 7（pwsh），没有则回退到系统自带的 Windows PowerShell 5.1；git-bash 用户可通过 `$SHELL` 沿用原 POSIX 环境。快捷键在 Windows 上映射为 Ctrl+Shift 系（对应 macOS 的 ⌘ 系）。

---

## 隐私与安全

**AI 能读取我的文件吗？**

默认**不能**。`cat` / `head` / `grep` / `less` 等读文件内容的命令在策略层直接被拒，这是结构性的，不是配置项。唯一例外：你主动把文件放进 `~/.kaya/`（trusted root），AI 才能用只读命令读它——脱敏仍然生效，写操作仍全拦。详见 [用户手册 · 让 AI 读取文件](docs/PRODUCT.md)。

**粘贴的密钥会被发出去吗？**

不会。你的输入在进入对话和日志前会自动脱敏（PEM 私钥整块 redact、API key、身份证号、手机号、信用卡等 30+ 类模式），命中时 UI 显示「已脱敏」标记。但脱敏不是绝对保证——不要把真正重要的密钥贴进任何 LLM 应用。

**云端模型会收到什么？**

只有你（经脱敏的）对话文本和（经脱敏的）工具结果，发往你在设置里选择的模型来源：

- **内置模型（默认）**：经你的登录令牌发往 yunhou 第一方服务器，由服务端持 key 代答；
- **OpenAI 兼容（自备 key）**：发往你选的 LLM 服务商（DeepSeek / OpenAI / OpenRouter 等）；
- **Ollama（本地）**：完全不外发。

你的本地文件、API key 内容、环境变量都不会被发送。本地 UI 状态、会话、审计日志全部只保存在你自己的机器上（`localStorage` + `~/.kaya/`），无云同步。登录状态下会收集**用量统计**（活跃天数、使用时长、平台与版本分布，v2.1.14 起）——**绝不含**对话内容、文件内容或个人身份信息，可随时在「设置 → 账户」关闭；未登录时无任何上报。

**API key 存在哪里？**

只存在你本机的**系统钥匙串**（macOS Keychain / Windows Credential Manager，v2.2.1 起；更早版本的本地配置会自动迁移），在你向所选 LLM 服务商发请求时随鉴权头发送，用于该服务商的鉴权。不会发给任何其他方，不写审计日志。kaya 不承诺「完全不上传」——任何联网请求都需要鉴权，鉴权就需要 key。

**AI 会删掉我的文件吗？**

AI 没有权限修改或删除你的文件。`rm -rf`、`sudo`、`dd`、`chmod 777` 等破坏性命令在安全关卡直接拦截；`curl | bash`、`brew install`、环境变量导出等也不会被执行。AI 执行的命令在独立子进程里跑，不经过你的终端；被拦时 UI 会显示具体拒绝原因。

**如何事后审计 AI 做了什么？**

每次会话自动落 JSONL 审计日志（`~/.kaya/audit/`）。Settings → 审计 可回看每个会话的完整事件流，并运行「规则回放 + 异常规则 + AI 评判」三层评估。

---

## 模型与配置

**需要自己的 API key 吗？**

不需要。默认走**内置模型**：微信扫码登录即可用，无需申请 key。也可以切到「OpenAI 兼容」用自己的 key（DeepSeek / Kimi / GLM / MiniMax / Qwen / OpenAI / OpenRouter / 硅基流动 / Ollama / 自定义端点等），或选 Ollama 完全本地推理。两级功能门禁（v2.1.5 起）：登录即可免费使用自备模型（BYOK）聊天；付费订阅解锁内置模型、coding 模式与 IM 渠道 / 远程访问。

**支持哪些模型服务商？**

BYOK 预设已从 5 个增加到 11 个（v2.1.14）：DeepSeek、Kimi（Moonshot）、GLM（智谱）、MiniMax、Qwen（DashScope）、硅基流动、OpenCode Zen、OpenAI、OpenRouter、Ollama、自定义（OpenAI 兼容）。默认模型跟随各厂商当前产品线，带思考能力的模型自动适配推理参数。v2.1.8 起支持保存多个模型配置（profile），顶栏模型切换器一键切换。

**思考模式（Thinking Mode）是什么？**

开启后 AI 先展示一段可折叠的推理过程再给答案，推理内容也会进审计日志。代价是更慢、token 消耗更多。注意：设置里的「Reasoning Model」字段目前仅供审计评估的 AI judge 使用，切换聊天实际用的模型请改「Model」字段。

**为什么命令被拒绝了？**

AI 尝试的命令可能：落在白名单之外（默认拒绝）、是破坏性命令、是交互式程序（vim / REPL）、或会修改系统（`sudo` / `brew install`）。UI 会显示具体拒绝原因。你在终端里自己输入的命令不受任何限制——那是你为自己负责的区域。

---

## 终端与标签页

**每个标签页是独立 shell 吗？**

是。每 tab 一条独立 PTY，加载你的 `.zshrc`（或 Windows 上的 PowerShell profile），环境变量、alias、conda 与原生终端一致。每 tab 有自己的工作目录、命令历史和滚动历史；关闭时历史自动合并回全局 shell 历史。

**常用快捷键？**

macOS：⌘T 新标签、⌘W 关闭、⌘1–9 切换、⌘D / ⌘⇧D 分屏、⌘⌥方向键切换窗格、⌘⇧F 聚焦、⌘0/=/- 字号。Windows 对应 Ctrl+Shift 系。完整列表见 [用户手册 · 快捷键](docs/PRODUCT.md)。

**标签页断开了怎么办？**

Settings → 通用 → Shell →「立即重连」。

---

## 编程 agent 协同（coding 模式）

**coding 模式是什么？**

打开顶栏的 coding 模式开关后，kaya 会主动看护你 tab 里运行的编程 agent（Claude Code / Kimi Code / OpenCode / Codex 等）：对方弹出确认时转达给你、任务跑完时总结进展。监控是只读的，决策始终由你做。

**AI 能直接操作我的终端吗？**

默认不能。AI 是终端的**观察者**（只读输出），不是操作者。唯一例外：coding 模式总闸开启 + 该 tab 前台是受支持的 AI CLI + 你在授权卡片上逐项确认，AI 才能向该 tab 代为输入。授权绑定当前进程，进程退出或 tab 关闭即失效，可随时撤销。你的普通 shell 提示符永远不接受 AI 输入。

**`/agents`、`@` 和 `/` 命令怎么用？**

对话输入框敲 `/` 唤出命令菜单：`/agents` 列出并绑定运行中的 agent，`/new` 新对话，`/sessions` 查历史，`/help` 帮助。也可以直接敲 `@`（v2.3.1 起）召唤运行中 agent 列表，选中即绑定并把 `@实例ID` 补全进输入框。绑定后，非内置的 `/命令`（如 `/auto`）会直接转发到该 agent 的输入行，不消耗 LLM 回合。

**Agents 管理中心能做什么？**

后台静默安装 / 更新 / 卸载编程 agent（npm > brew > 官方脚本，失败自动换下一种方式）。模型配置统一在 **Settings → Models** 的模型库里维护（API key 只存系统钥匙串），Agents 面板里点「使用」把条目绑定到某个 agent、写入它自己的配置文件（写前自动 .bak 备份），之后编辑 / 删除条目会自动同步到已绑定的 agent。也可以在对话里直接说「帮我装一下 Claude Code」。

---

## IM 渠道（飞书 / Slack / 企业微信）

**能在飞书 / Slack 里用 kaya 吗？**

可以（飞书 v2.1.0 起，Slack v2.3.6 起）。Settings → IM 渠道：飞书推荐用「扫码接入」一键完成配置；Slack 在 api.slack.com 自建应用（开启 Socket Mode、订阅 message 事件）后把两个 token 填进来即可。启动后在 IM 里给机器人发消息，由你本机的 kaya 处理并流式回复——长连接直连，无需公网 IP。企业微信入口当前置灰「即将支持」（存量配置保留，后续版本恢复）。

**陌生人能给机器人发消息吗？**

不能。准入 allowlist 默认拒绝名单外的一切消息；名单为空时渠道无法启动。IM 消息与桌面聊天走同一条五道关卡隐私管线，平台凭据只存系统钥匙串；IM 里的批准 / 拒绝卡片与桌面授权卡互认。

---

## 远程访问（浏览器）

**人不在电脑前能用 kaya 吗？**

可以（v2.3.2 起）。Settings → Remote Access 打开开关（需登录 yunhou 账号且有有效订阅），然后在浏览器登录 yunhou 官网、打开「Remote Control」页面，即可远程查看和操作本机 kaya——会话、运行中 agent 状态、审批提示与桌面实时同步。

**远程访问的数据安全吗？**

该功能默认关闭、显式 opt-in，关闭立即断开。会话内容经 yunhou 服务器**仅内存转发、不持久化**；**API key 绝不经过中继**——本机仍直连模型服务商；全程 TLS 加密。远程回合走与桌面相同的五道关卡，写操作与桌面共享同一份授权状态（互相认可）。

---

## 故障排查

**AI 一直不响应 / 回复很慢？**

- 流式响应静默 30 秒会自动超时；可按 Esc 或 ⏹ 手动中断。
- 开启思考模式会增加延迟；检查网络，或在设置里换更快的模型。
- 首次连接 LLM 最长 30 秒握手超时，报「LLM 连接超时」时检查网络与 API key 后重试（v2.1.11 起弱网会自动重试）。

**终端无响应？**

按 Esc 中断当前回合 → Settings → 通用 → Shell →「立即重连」→ 仍不行就 ⌘W 关掉该标签页再 ⌘T 新建。

**会话丢失？**

检查 `~/.kaya/conversations/` 是否被清理。要清除全部本地数据：`rm -rf ~/.kaya/`。

**诊断日志在哪？**

Settings → App 日志 →「打开日志文件夹」，日志在 `~/.kaya/logs/`（不含你的对话内容）。

**内置模型报「Request Rejected by Upstream (502)」怎么办？**

这是上游模型服务拒绝了请求（常见原因：上下文过长 / 内容审查 / 额度不足）。kaya 会先**自动压缩早期对话历史并重试一次**（只影响 AI 的上下文，你看到的聊天记录不变）；仍失败时错误卡片会给出对应原因的处理指引，并附「新会话」按钮一键开新对话（不影响终端标签页和运行中的 agent）。

**还有其他问题？**

在本仓库提 [Issue](../../issues)。
