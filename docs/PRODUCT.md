# kaya 用户手册 / User Guide

> **Mission Control for AI Coding Agents · Privacy First · Fast · Simple & Effective**

kaya 是一款运行在你 Mac 上的 AI 终端:左侧是你真实的 zsh,右侧是与 AI 的对话窗口。AI 通过受限的诊断工具协助你处理日常开发、调试、环境检查和命令解释。

**语言 / Language**

- [中文用户手册](#中文用户手册)
- [English User Guide](#english-user-guide)

---

## 中文用户手册

### 欢迎使用 kaya

kaya 是一款 macOS 桌面应用,把 AI 助手直接接到你日常使用的终端里。打开应用,你会看到左侧是一块和你熟悉的 Terminal、iTerm2 一样的终端面板,右侧是一个 AI 对话窗口。两个面板共享同一个 shell 会话,你在终端里看到什么,AI 也能看到,反过来你也可以让 AI 帮你解释、查询或操作。

kaya 的目标是让你在**不离开真实 shell**的前提下获得 AI 协助:你能继续使用自己的 zsh 别名、自己的脚本、自己熟悉的快捷键,AI 不会把你困在一个沙箱里。

> 注意:AI 不会接管你的整个终端。AI 默认只读你的终端输出;只有当你**打开顶栏的 coding 模式开关**、且在对话中的授权卡片里明确允许、并且该 tab 运行的是受支持的 AI CLI(Claude Code / Kimi Code / Reasonix / OpenCode)时,AI 才能代为向该 tab 输入。coding 模式开关是「驾驶」的总闸门:开关关闭时,对话端连向终端 AI 发输入的能力都不存在(该工具不注册);打开之后仍需要每个 tab 单独的授权卡确认。授权绑定当前进程,进程退出或 tab 关闭即失效;随时可点 tab 上的 🤖 角标撤销。你的普通 shell 提示符永远不接受 AI 输入。同理,AI 帮你修改编程 agent 的配置文件(如 Claude Code 的 settings.json)时,每一次修改也都必须经你在授权卡片上明确允许才会落盘,授权只对当前会话有效。

---

### 产品能做什么

kaya 适合这些日常工作:

- **代码解释**:贴一段报错、一段 shell 脚本、一个正则,让 AI 解释它在做什么
- **环境检查**:查看当前系统信息、磁盘使用、运行中的进程、网络端口等
- **文件查询**:列出目录内容、统计文件大小、查找匹配的文件名
- **命令构造**:让 AI 帮你拼出 `find` / `awk` / `sed` 等组合命令,你审阅后再执行
- **日志和错误解读**:让 AI 解释一段错误输出,定位可能的原因

kaya **不适合**也不试图做的事:

- 自动改写你 Mac 上的源代码
- 在没有你确认的情况下执行破坏性操作(如删除文件)
- 在你的终端里跑交互式程序(如 `vim`、REPL)
- 在没有你的允许下访问你 Mac 上的任意文件内容

AI 给出的命令始终需要你在终端里**自己看到、自己执行**。这是设计的一部分:AI 是协助者,你始终是决策者。

---

### 隐私与安全边界

隐私是 kaya 的**第一优先级**。下面是默认情况下你能期待的边界。

**默认 AI 看不到的内容**

- 你 Mac 上的任意文件**正文**:`cat`、`head`、`grep`、`less` 等读取命令默认不可用
- 你 Mac 上的系统环境变量:`env`、`printenv` 默认不可用
- 你终端里你自己输入的命令和输出——这部分是**完全属于你的**,AI 不拦截也不审核

**默认 AI 可以查询的**

- **元数据**:目录列表、文件大小、修改时间、当前用户名、当前系统版本等
- 你终端里当前的输出内容(便于你让 AI 解释一段报错)
- 你的任务清单(只读,AI 可查看但不能修改)
- 你主动复制到对话里的文本

**`~/.kaya/` 可信目录**

如果你需要 AI 帮你读取某个文件的内容,把文件放进 `~/.kaya/` 目录下,然后告诉 AI 文件路径。这是唯一允许 AI 读取文件正文的位置。详见 [让 AI 读取文件](#让-ai-读取文件)。

**脱敏是防线,但不是绝对保证**

在你输入文本、AI 读取文件、工具执行返回结果时,kaya 会自动识别并替换掉常见的密钥、PEM 私钥、身份证号、手机号、信用卡号等敏感信息。脱敏有 30 多类规则,涵盖常见场景。但 kaya 不承诺能识别**所有**可能的秘密——你不要把真正重要的密钥贴进任何 LLM 应用。

**云端 Provider 会收到什么**

当你在对话里发送消息、让 AI 调用工具时,经过脱敏的对话文本和工具结果会发送到你**在设置里选择的模型来源**:

- **内置模型(默认,yunhou `/chat` 代理)**:脱敏后的内容经你的登录令牌(JWT)发往 **yunhou 第一方服务器**,由服务端持 DeepSeek key 代答。kaya 客户端在发送前会按服务端限额裁剪(消息条数 / 每条字节 / 总字节上限,超限丢弃最早的历史消息)。登录令牌(refresh_token)只存在 macOS 钥匙串。
- **OpenAI 兼容(自备 key)**:脱敏后的内容发往你选择的 LLM 服务商(DeepSeek / OpenAI / OpenRouter 等),随你的 API Key 鉴权。
- **Ollama(本地)**:完全不外发(见下)。

kaya 不会把你的内容发给上述路径之外的任何其他第三方。本地 UI 状态、会话、审计和应用日志都只保存在你自己的 Mac 上。

**API Key 怎么用**

你填在设置里的 API Key **只保存在你本机的应用设置中**,在你向所选 LLM 服务商发起请求时随鉴权头发送,用于该服务商的鉴权。kaya **不会**把你的 API Key 发给任何其他方。**不**承诺"完全不上传"——任何联网请求都需要鉴权,鉴权就需要 key。

**离线推理: Ollama**

如果选择 Ollama 作为 Provider,所有推理完全在你的 Mac 上进行,**不会**有任何外发请求。适合对隐私最敏感的场景。

**五道保护概览**

简单来说,你输入到输出的每一句话都经过:

1. **输入保护**:粘贴的密钥和长文本被脱敏和截断
2. **命令策略**:可执行命令被白名单限制,默认拒绝未知命令
3. **安全检查**:破坏性、交互式、系统修改类命令被拦截
4. **隔离执行**:AI 的工具调用在独立子进程中运行,不影响你的终端
5. **输出脱敏**:工具输出再脱敏一次才进入对话

---

### 安装 kaya

**主路径:下载 DMG**

1. 打开 kaya 官网或 GitHub Release 页面,下载 `kaya-x.y.z-macos-universal.dmg`(Apple Silicon 和 Intel 通用)
2. 双击 DMG 挂载,会看到 `kaya.app` 窗口
3. 把 `kaya.app` 拖入 `/Applications` 文件夹
4. 在 Launchpad 或 `/Applications` 里打开 kaya

**首次启动**

如果 macOS 提示无法验证应用:

1. 在 Finder 里找到 `/Applications/kaya.app`
2. 在系统设置的 **隐私与安全性** 中允许已确认的 kaya 应用
3. 再次打开 kaya

不要关闭 Gatekeeper,也不要执行来源不明的提权命令。

> 当前官网用户手册只推荐 DMG 安装。命令行版本的预编译 release 已暂停提供；需要 CLI 的高级用户应参考独立 CLI 文档。

---

### 首次配置

第一次打开 kaya 后,先做以下配置。

**方式 A(默认,免 API Key):内置模型 + 微信登录**

kaya 出厂默认走**内置模型**(yunhou `/chat` 代理,服务端持 DeepSeek key 代答),无需自己申请 API Key,只需登录 yunhou 账号:

1. 点击右上角 **⚙ 设置**,或在对话窗口点 **微信扫码登录**(顶栏账户图标 / 登录按钮)
2. 系统浏览器打开微信扫码页 → 扫码 → 自动跳回 kaya 完成登录(令牌写入 macOS 钥匙串)
3. 回到对话窗口即可直接提问。订阅/试用状态在 **⚙ 设置 → 账户** 标签页查看

**方式 B(自备 API Key):OpenAI 兼容**

如果你想用自己的 LLM key:

1. 打开 **⚙ 设置**,在 **通用** 标签页的 **LLM** 小节,把 **模型来源** 切到 **OpenAI 兼容**
2. 选择 **Provider**(服务商):
   - **DeepSeek**:国内常用,默认模型 `deepseek-v4-flash`
   - **OpenAI**:国际通用,需要海外访问
   - **OpenRouter**:聚合多家模型,可在一个 key 下切换模型
   - **Ollama**:本地推理,需要先启动 `ollama serve`
   - **自定义(OpenAI 兼容)**:填入任意 OpenAI 兼容端点的 Base URL 和模型
3. 填入 **API Key**:从你选择的 Provider 官网获取
4. 检查 **Base URL**:选择预设后会自动填入,通常无需修改
5. 选择 **Model**:默认模型对大多数任务已够用
6. 根据需要开启 **Thinking Mode**(思考模式):让 AI 在回答前先展示推理过程
7. 点击保存,回到对话窗口

第一次配置完成后,kaya 会自动记住。下次打开无需重新设置。

---

### 完成第一次对话

配置好后,在右侧对话窗口底部输入框里输入:

```
查看当前系统版本和 CPU 架构
```

按 **Enter** 发送。你会看到:

1. AI 先展示一段折叠的思考过程(如果开启了 Thinking Mode)
2. 然后出现一张**工具卡片**,显示 AI 调用了哪个工具、传了什么参数
3. 卡片状态从 **运行中** 变成 **已完成**,展开可看完整 stdout / stderr
4. AI 把结果整理成自然语言回答

**识别三种元素**

- **思考区**:可折叠的灰色背景段落,展示 AI 的推理过程
- **工具卡片**:每次 AI 调用工具都会出现,显示工具名、参数、运行状态、输出
- **最终回答**:AI 的自然语言结论

如果结果不是你想的,可以继续追问:

```
磁盘空间使用情况呢?只关心当前用户的文件
```

如果 AI 卡住或答得不好:

- 按 **Esc** 或点击 **⏹** 按钮中断当前回合
- 然后换个问法重新开始

---

### 对话内 slash 命令

在对话输入框敲 **`/`** 会唤出命令菜单(键盘上下选择或鼠标点击补全),也可以用 `/help` 随时列出所有命令说明。内置命令:

- **`/agents`**:列出当前正在运行的协同 agent(kimi / claude / opencode / reasonix 等)。点击卡片把它绑定为**本会话**的转发目标——绑定是会话级的,新对话自动清空
- **`/new`**:新建对话(与顶栏的新建按钮是同一入口)
- **`/sessions`**:从审计记录列出历史会话,点击卡片查看只读的历史内容
- **`/help`**:列出全部命令及说明

以 `/` 开头但不是内置命令的输入(如 `/auto`)不会报错:如果当前会话已绑定 agent,kaya 会把原文写入该 agent 所在终端的输入行并回车——相当于替你敲给它。写入前做 fail-closed 复验(pane 仍存活、前台仍是绑定的那个 agent、不在等待菜单中),任一不满足就不写入。这类转发**不消耗 LLM 回合**,转发记录会以卡片形式进入消息流并记入审计。跨 tab 换绑时,kaya 会向新绑定的 agent 注入一条一次性上下文便签,让它知道之前的来龙去脉。

---

### 任务清单(Task)

kaya 内置一个轻量任务清单,帮你和 AI 一起跟踪要做的事:

- 顶栏的待办开关打开 **任务面板**,可以查看、勾选、管理当前任务
- 在对话里也能直接用自然语言管理:让 AI 从对话中**提炼**任务,或**查看 / 添加 / 修改 / 删除 / 清空**清单
- AI 侧只有一个**只读**工具(查看任务清单),用来对齐上下文——它不能替你改动清单

---

### 使用终端与标签页

**基本终端操作**

左侧是你真实的 zsh。常用操作:

- 输入命令后按 **Enter** 执行
- 鼠标选中文字按 **⌘C** 复制,**⌘V** 粘贴
- **⌘A** 全选当前行
- 滚动条或触摸板向上滚动查看历史输出

你在终端里输入的命令**完全属于你自己**,AI 不拦截也不审核——这是你为自己负责的区域。

**多标签页**

kaya 支持同时打开多个终端,每个标签页是独立的 shell 会话:

| 操作 | 快捷键 |
|---|---|
| 新建标签页 | **⌘T** |
| 关闭当前标签页(焦点在终端时) | **⌘W** 或标签上的 × 按钮 |
| 切换到第 N 个标签页 | **⌘1** 至 **⌘9** |
| 上一个标签页 | **⌘⇧[** |
| 下一个标签页 | **⌘⇧]** |
| 重排标签页(焦点在标签栏时) | **⌥⌘←** / **⌥⌘→** |

每个标签页有自己的工作目录、命令历史和滚动历史。关闭标签页时,标签页内的命令历史会自动合并到全局 `~/.zsh_history`。

**字体大小**

- **⌘=** 或 **⌘+**:放大 1px
- **⌘-**:缩小 1px
- **⌘0**:重置为默认大小

**关闭确认**

关闭标签页(**⌘W** 或 × 按钮)和关闭整个 kaya(左上角红色 ×)都会弹出确认对话框,防止误操作丢失未保存的工作。

**⌘Q 不弹确认**

使用 **⌘Q** 退出时,macOS 不显示确认对话框,但 kaya 会先尝试把本地各标签页的 shell 历史合并到 `~/.zsh_history`,然后再退出。尽量别用 ⌘Q,推荐红 X 或菜单退出。

**断开的标签页**

如果标签页因为网络或进程异常断开:

- 标签状态显示为灰色
- 打开 **⚙ 设置** → **通用** 标签页的 **Shell** 小节,点击「立即重连」重连该会话对应的 PTY

**tab 等待确认提醒**

当某个 tab 里运行的 AI CLI(kimi code / claude code 等)弹出需要你决策的确认(如命令审批)时,kaya 会主动感知并把决策收口到对话输入框,而不需要你自己盯着终端:

- 对话窗口出现一张通知卡片,显示哪个 tab 的哪个 AI 在等待、确认内容摘要;卡片上只有「忽略」(确认内容变化会重新提醒)
- 对应 tab 上显示一个提醒角标
- 最近一张待决的确认如果带选项菜单,会以决策条常驻在对话输入框上方:显示决策文案、各序号对应的含义,并提示「输入数字选择」
- 你直接在对话输入框敲菜单序号并发送,kaya 会把对应按键写回该 tab 里的 AI CLI。写回前会做多重复验:tab 仍存活、前台进程仍是受支持的 AI CLI、该菜单仍显示在画面中、序号在合法范围内——全部通过才允许写入,且只接受菜单上的个位数字序号
- 序号越界、输入的不是裸数字、或当前没有待决确认时,输入不会被拦截,照常被当作普通消息发送
- 你在终端里处理完后,卡片自动变为「已处理」

kaya 永远不会替你回答 AI CLI 自己的确认——那是它自身的安全闸,决策始终由你本人做出。

**coding 模式(主动看护编程 agent)**

打开顶栏的 coding 模式开关后,kaya 会主动盯着你运行在 tab 里的编程 agent(Claude Code / Kimi Code / Reasonix / OpenCode 等),不需要你自己盯着终端:

- 检测到对方**弹出确认、等待你决策**时,kaya 自动发起一个对话回合,转达对方需要什么决策并询问你的指示
- 检测到对方**任务跑完(画面空闲)**时,kaya 主动读取结果、总结进展并询问下一步
- 监控是**只读**的:kaya 只报告和询问,永远不会替你点确认——决策始终由你做
- 这个开关同时是「驾驶」的**总闸门**:只有开关打开,对话端才具备向编程 agent tab 发指令的能力(仍需每个 tab 单独的授权卡确认);开关一关,该能力**从下一回合起**整个下线(工具不再注册;正在进行的回合按回合开始时的状态走完),监控不受影响的部分只剩被动的通知卡片与 tab 角标
- 自动回合携带的确认内容摘要经过脱敏;每个 tab 有 60 秒冷却,对话进行中不打扰(回合收尾后补发)
- 注意:coding 模式的自动回合会随对话发送给你当前选用的模型(与手动提问同一条外发路径);关闭开关即完全不触发

**Agents 管理中心**

应用菜单 → **Agents**(或点顶栏右侧的 Agents 图标)打开管理中心。安装、更新、模型配置也可以在对话里直接让 kaya 做(比如说「帮我装一下 Claude Code」)——两条路径背后是同一套引擎:

- **后台安装 / 更新**编程 agent:安装前自动选择一个可用方式(npm > brew > 官方脚本),进度实时显示在卡片上;失败后不会自动降级换方式,可改用「在终端安装」手动兜底
  - npm 方式遇到网络类失败会自动换 npmmirror 镜像**重试一次**;brew 与官方脚本没有重试
  - 安装没有总超时:长时间无动静时,看面板任务卡片的输出尾部(保留最后 50 行)判断是慢还是卡住
  - 暂不支持卸载 script(官方脚本)装的 agent——请按官方文档手动卸载;npm/brew 装的支持卸载(带归属校验,不是该管理器装的不卸),且只移除程序本体,配置与历史数据保留
- **模型供应商库**(cc-switch 式):为每个 agent 维护多套供应商配置(base URL / model / API Key),点「使用」即写入该 agent 自己的配置文件(自动备份 .bak)
  - 对话里 kaya 改配置前会弹授权卡片,你确认后才落盘;密钥值不会回显在对话里
  - 恢复旧配置:手动还原写入前的 `.bak` 备份(如 `~/.claude/settings.json.bak`)
  - 注意:供应商的 API Key **明文**保存在 `~/.kaya/agent-providers.json`(与 cc-switch 一致),请勿与他人共享该文件
- 通过 IM 渠道(飞书/企微)远程使用时:仅支持读/改配置(同样有授权卡);查状态、安装、更新在 IM 里不可用

---

### 让 AI 读取文件

默认 AI 不能读取你 Mac 上的文件。如果你想让 AI 帮你分析一段代码、查看一个日志、解释一个配置文件,需要先把文件放进可信目录。

**步骤**

1. 在终端里创建或使用 `~/.kaya/` 目录:

```bash
mkdir -p ~/.kaya/my-project
cp ~/my-project/main.ts ~/.kaya/my-project/
```

2. 在对话里告诉 AI 文件路径:

```
读取 ~/.kaya/my-project/main.ts 并解释它做了什么
```

3. AI 会调用允许的读取命令(如 `cat` 或 `head`)读取文件内容,展示给你

4. 用完后删除不再需要的副本:

```bash
rm -rf ~/.kaya/my-project
```

**允许的读取命令**

在 `~/.kaya/` 下,以下命令可用:

- `cat` / `bat` / `view`:读取文件全部内容
- `head` / `tail`:读取文件开头或结尾部分
- `grep` / `awk` / `sed -n`:在文件内容中搜索
- `wc` / `sort` / `uniq`:统计和排序
- `strings` / `xxd` / `hexdump`:查看二进制内容

**不允许的操作**

即使在 `~/.kaya/` 下,以下操作仍然被拦截:

- 修改文件(`sed -i`、重定向、`mv` 等)
- 删除文件
- 执行文件内容

AI **只能读,不能改**。这是设计如此。

**下载文件**

AI 也能从网络下载文件到 `~/.kaya/`(例如 `curl -o ~/.kaya/...`)。危险的文件类型(如 `.sh`、`.exe`、`.dmg`)会被拦截。

**提醒**

- 不要因为有脱敏就主动把密钥放进 `~/.kaya/` 给 AI "看看"
- 用完后及时删除 `~/.kaya/` 下不再需要的文件
- 链接到 `~/.kaya/` 之外的符号链接不会被识别为可信

---

### 使用联网搜索

kaya 支持通过 Tavily 让 AI 联网搜索最新信息。Tavily 是可选功能,默认关闭。

**启用步骤**

1. 打开 [tavily.com](https://tavily.com) 注册并获取 API Key
2. 在 kaya 里打开 **⚙ 设置** → **联网搜索**
3. 填入 Tavily API Key
4. 点击保存

**使用**

启用后,在对话里让 AI 搜索:

```
搜索一下 React 19 的最新特性
查一下 DeepSeek 最新模型的消息
搜索并总结这篇英文文章的主要内容
```

AI 会调用搜索工具获取结果,整理后告诉你。

**隐私说明**

Tavily 是**唯一**会把你(经脱敏后的)问题文本发给第三方的功能。Tavily 收到的只是你问 AI 的问题文本,不是你的文件、不是你的 API Key。**关闭 Tavily 即可关闭这条外发路径**。

**可信度提醒**

搜索结果是来自互联网的**不可信输入**——可能是过时、错误或带有偏见的内容。AI 给出的总结仍然需要你核实关键事实。开启 Tavily 不会让 AI 跳过内容保护,搜索得到的网页内容在进入模型上下文前也会经过脱敏。

---

### 通过飞书 / 企业微信使用 kaya

v2.1.0 起,kaya 支持把 IM 消息接到本机:在飞书或企业微信里给机器人发消息,由本机 kaya 的 LLM 处理并**流式回复**——不用打开 kaya 窗口,在 IM 里就能用。

**开启路径**

1. 打开 **⚙ 设置** → **IM 渠道** 标签页(也可从系统菜单 kaya → **IM 渠道…** 直达),选择平台——当前版本**企业微信入口置灰「(即将支持)」**,仅飞书开放接入(存量企微配置保留,后续版本再开放)
2. **推荐:点「飞书扫码接入」**(仅飞书平台提供):用手机飞书扫描二维码并确认,kaya 自动创建应用并回填 App ID 与创建者 open_id 到 allowlist;App Secret 直接写入 **macOS 钥匙串**,不落盘、不回读显示
   - 手动配置(fallback):自行到[飞书开放平台](https://open.feishu.cn)或企微管理后台创建应用后,填入飞书 App ID + App Secret,或企微 BotID + Secret
3. 配置**准入 Allowlist**:每行一个用户 ID(飞书填 open_id「ou_」开头,企业微信填成员 userid);扫码接入会自动锁定创建者本人
4. 点击**启动渠道**,状态显示「● 运行中」即可在 IM 里发消息。**如提示无权限**,请到飞书开放平台为该应用开通 im:message 相关权限并发布版本

**要点**

- **长连接直连平台**:飞书走官方 WebSocket 长连接,企微走官方 openws 通道,**不需要公网 IP 或回调服务器**
- **allowlist 默认拒绝陌生人**:名单外的用户消息直接丢弃,不进 LLM;名单为空时渠道无法启动
- **同一套隐私防线**:IM 消息与桌面 chat 走同一条五道关卡 pipeline(输入脱敏 → 命令白名单 → 安全检查 → 隔离执行 → 输出脱敏);IM turn 的工具表与桌面全集对齐(可读终端 / 任务清单),`terminal_write` 仍由 coding 模式总闸照管——总闸为桌面与渠道共享的原子标志,运行中在桌面扳动开关即时写入渠道侧,下一 IM turn 生效
- **桌面 → IM 镜像(opt-in,默认关)**:开启后桌面 chat 的消息流实时推送到你指定的 IM 聊天,你的输入以「你:…」前缀文本回显;IM 侧回复也同步出现在桌面聊天面板。新建桌面会话时镜像自动迁移并推一条「已开启新会话」通知。开关入口有两处——Settings 的「镜像桌面聊天到飞书」checkbox 与顶栏 coding 旁的镜像 icon,是同一个开关,点击即时生效
- **飞书授权卡片**:IM turn 触发 terminal_write / agent 配置修改授权时,飞书里直接收到带「批准 / 拒绝」按钮的卡片,决议与桌面授权卡共享同一份授权状态(互认);操作者必须在准入 allowlist 内。批准后 kaya 会**自动补一个回合**,请 AI 重试刚才被暂停的操作——不用再手动发一句话把它唤醒
- **agent 状态主动通知**:镜像开启时,编程 agent 等输入 / 跑完空闲 / 上线 / 退出会主动推送到镜像聊天(按渠道语言出中文 / 英文,同类事件有冷却不刷屏)
- 会话与审计照常落 `~/.kaya/`;LLM 使用「通用」页配置的模型来源
- **如实说明(企微限制)**:企业微信的镜像回显与主动通知依赖活跃回调,纯桌面场景(无 IM 对话在途)可能静默丢失;本轮仅飞书做了真机验证

隐私细节详见 [docs/PRIVACY.md](./PRIVACY.md) 的「IM channel 入口」与「IM 渠道与镜像」两节。

---

### 使用思考模式

**什么是思考模式**

开启后,AI 在给出最终回答前,会先进行一段**链式推理(Chain-of-Thought)**:把它的思考过程以可折叠的段落展示给你,然后再回答你的问题。

**为什么使用**

- **可解释**:你能看到 AI 为什么得出这个结论
- **可审计**:推理过程会单独记录,事后可回放
- **更准确**:对于复杂问题,带推理的回答通常比直接回答更准确

**代价**

- **更慢**:AI 需要时间思考再回答
- **更贵**:消耗的 token 更多,API 费用更高
- **可能更长**:推理内容会占一定篇幅

**怎么开启**

打开 **⚙ 设置** → **通用** 标签页的 **LLM** 小节,找到 **Thinking Mode**,打开开关即可。

**切换推理模型**

kaya 设置里有一个 **Reasoning Model** 字段。

> **重要**:**Reasoning Model** 字段对**聊天请求**而言**仅作显示用途**,它不会切换 AI 实际请求的模型 ID。该字段目前仅审计评估的 **AI judge** 使用(用它选择评判模型)。

要切换 AI 使用的模型,请改设置里的 **Model** 字段。开启思考模式后,kaya 会按所选 Provider 的规则自动适配推理参数(例如 DeepSeek 会注入 `reasoning_effort` 参数),**不需要**单独切换一个"推理专用模型"。

如果你看到 Reasoning Model 字段,把它当作一个**预留**字段理解即可,它的值不会影响聊天请求。

---

### 管理会话与历史

**会话持久化**

kaya 自动保存你的对话。三个层级自动同步:

- UI 消息(左侧对话窗口)保存在应用本地
- LLM 上下文(供 AI 回顾)保存在 `~/.kaya/conversations/`
- 审计日志(详细事件流)保存在 `~/.kaya/audit/`

刷新应用或重启 Mac 后,对话和设置都会自动恢复。

**新对话**

点击右上角 **↻ 新对话** 按钮:

1. 停止当前正在进行的回合(如果有)
2. 清空当前 UI 消息
3. 生成新的会话上下文
4. 开始一个全新的对话

**查看历史会话**

**新对话** 是一个纯图标按钮,旁边没有历史列表。查看历史的入口有两个:在对话输入框用 **`/sessions`** 命令列出历史会话并点卡片查看(**只读**,不能恢复上下文继续聊);或打开 **⚙ 设置 → 审计** 查看完整事件流。

**清除数据**

kaya 没有逐会话删除的 UI。要清除所有本地数据,在终端运行:

```bash
rm -rf ~/.kaya/
```

**会话标签页、退出行为**

- 关闭标签页(**⌘W**):弹出确认对话框;确认后,关闭该标签页,合并该标签页的 shell 历史到 `~/.zsh_history`
- 关闭整个 kaya(红 X):弹出确认对话框;确认后,合并所有活跃标签页的 shell 历史到 `~/.zsh_history`,然后退出
- **⌘Q**:不弹确认对话框;kaya 先尝试合并本地历史到 `~/.zsh_history`,然后退出。如非必要,请优先使用红 X 退出

---

### 查看审计记录

kaya 每次会话都自动记录完整的审计日志,你可以随时回看 AI 做了什么。

**打开审计页面**

**⚙ 设置** → **审计**。

**会话列表**

左侧列出所有会话,按时间排序。点击任一会话查看其事件流。

**事件类型**

- 会话开始 / 结束
- 你的每次输入(含脱敏结果、原始字节数)
- AI 每次工具调用决策(为什么允许或拒绝)
- 工具调用完成(含 stdout、stderr、退出码)
- AI 的每次推理片段
- 错误事件

**运行评估(Evaluate)**

点击会话右侧的 **Evaluate** 按钮,kaya 会做三轮复审:

1. **规则回放**:用当前版本的策略,重放所有工具调用决策。任何与当前规则不一致的地方都会被标记
2. **异常启发式**:检查 5 类异常模式 —— "文件读取未脱敏"、"网络输出未脱敏"、"环境变量输出未脱敏"、"工具调用前缺少推理片段"、"Tavily 大输出未脱敏"
3. **AI 评判**:由独立 AI(始终带思考模式)对整个会话做最终复审

**审计日志 vs App 日志**

- **审计日志**:每次会话的 LLM 相关事件,用于事后回放
- **App 日志**:跨会话的应用诊断日志,用于排查 kaya 本身的运行问题

两者独立,不要混淆。

---

### 设置参考

打开 **⚙ 设置**,标签页为:**通用 / IM 渠道 / 审计 / App 日志 / 快捷键 / 账户**(dev 模式另有 Debug)。**LLM、联网搜索、Shell、语言、脱敏** 等都是 **通用** 标签页内的小节,不是独立标签页。

**IM 渠道**

把飞书 / 企业微信消息接到本机 kaya:配置平台凭据(存 macOS 钥匙串)、准入 allowlist,启动 / 停止渠道并查看运行状态。详见 [通过飞书 / 企业微信使用 kaya](#通过飞书--企业微信使用-kaya)。

**通用 → LLM 小节**

| 项 | 说明 |
|---|---|
| 模型来源 | **内置模型**(yunhou 代理,免 key,默认)/ **OpenAI 兼容**(自备 key) |
| Provider | LLM 服务商(仅 OpenAI 兼容模式) |
| API Key | 服务商密钥,仅存本机(仅 OpenAI 兼容模式) |
| Base URL | API 端点,选择预设后自动填入(仅 OpenAI 兼容模式) |
| Model | 对话主模型(仅 OpenAI 兼容模式) |
| Reasoning Model | 显示字段(目前仅审计评估的 AI judge 使用,详见思考模式章节) |
| Thinking Mode | 启用思考模式 |
| Scrub Level | 脱敏强度(aggressive / balanced / strict),默认 aggressive |

> 注:**Max Context Messages** 控制 Rust agent loop 的 `conversation::trim_history`(保留 system prompt + 最近 N 条消息)。2026-08-05 review C1 接线(`agent.rs::reload_trimmed`)。持久化上限 200 条/会话(`conversation.rs::MAX_MESSAGES_PER_CONVERSATION`)。

**通用 → 其它小节**

| 项 | 说明 |
|---|---|
| Language | 界面语言 |
| Shell | PTY 使用的 shell 路径 |
| 保留终端滚动历史 | 重启后是否保留各标签页的终端历史 |
| 终端字体大小 | 终端显示的字号 |
| 联网搜索 | 填入 Tavily API Key 并启用联网搜索 |

**账户**

查看登录状态、订阅/试用、刷新订阅、登出;高级里可手动粘贴登录令牌兜底。

**审计**

查看历史会话、运行评估。

**App 日志**

查看诊断日志、**手动**清理 7 天前的旧日志。

**关于**

「关于」不是设置里的标签页。通过**系统菜单**(macOS 菜单栏 kaya 菜单 → 关于 kaya)打开独立的关于对话框,查看版本号、构建信息、许可证;**「检查更新…」也在系统菜单**里。系统菜单还提供 **IM 渠道…** 与 **快捷键…** 两个快捷入口,分别直落设置的对应标签页。

---

### 快捷键

**标签页**

| 快捷键 | 行为 |
|---|---|
| **⌘T** | 新建标签页 |
| **⌘W** | 关闭当前标签页(焦点在终端时);关闭整个 kaya(焦点不在终端时) |
| **⌘1** – **⌘9** | 切换到第 N 个标签页 |
| **⌘⇧[** | 上一个标签页 |
| **⌘⇧]** | 下一个标签页 |
| **⌥⌘←** / **⌥⌘→** | 重排标签页(焦点在标签栏时) |

**分屏**

| 快捷键 | 行为 |
|---|---|
| **⌘D** | 垂直分屏(左右) |
| **⌘⇧D** | 水平分屏(上下) |
| **⌘⌥←→↑↓** | 在窗格间移动焦点 |

**聚焦**

| 快捷键 | 行为 |
|---|---|
| **⌘⇧F** | 聚焦到上次活跃窗格 / 退出聚焦(同右上角 ⛶ 按钮) |

**终端字体**

| 快捷键 | 行为 |
|---|---|
| **⌘=** / **⌘+** | 放大 1px |
| **⌘-** | 缩小 1px |
| **⌘0** | 重置为默认大小 |

**终端内编辑**

| 快捷键 | 行为 |
|---|---|
| **⌘C** | 复制选中文字 |
| **⌘V** | 粘贴 |
| **⌘A** | 全选 |

> 在 macOS 上,**Ctrl + 字母键**(例如 Ctrl+W、Ctrl+A)会**直接传给终端**,用于 vim、readline、bash 等命令行工具的快捷操作。

**对话**

| 快捷键 | 行为 |
|---|---|
| **Enter** | 发送消息(中文输入法未激活时) |
| **Esc** | 中断 AI 当前回合 |

**关于 ⌘W**

⌘W 的行为取决于焦点位置:

- 焦点在终端输入框:关闭当前标签页
- 焦点不在终端(对话输入框、设置页面等):关闭整个 kaya

---

### 常见问题

**AI 能不能读取我的文件?**

默认不能。`cat`、`head`、`grep` 等命令默认被拦截。如果你需要 AI 读取文件,把文件放到 `~/.kaya/` 下,然后告诉 AI 路径。详见 [让 AI 读取文件](#让-ai-读取文件)。

**粘贴的密钥会被发出去吗?**

不会。你输入的密钥在 kaya 内部会被自动脱敏(PEM 私钥整块、API Key、身份证号、手机号、邮箱、信用卡等常见模式),脱敏后的文本才进入对话和日志。被替换的部分在 UI 上会有"已脱敏"标记。

**API Key 安全吗?**

API Key 仅保存在你本机的应用设置中,在你向所选 LLM 服务商发起请求时随请求头发送,用于该服务商的鉴权。**不会**发送给任何其他第三方。**不**承诺"完全不上传"——任何联网请求都需要鉴权,鉴权就需要 key,这是联网请求的基本机制。

**云端模型会收到什么?**

云端 Provider 会收到:你输入的(经脱敏的)对话文本、AI 调用工具后的(经脱敏的)工具结果。**不会**收到:你的本地其他文件、你的 API Key 内容、你的环境变量。Provider 是你在设置里选择的那一家(DeepSeek、OpenAI 等)。

**怎么用本地模型?**

选择 Ollama 作为 Provider,确保先在终端启动 `ollama serve`,然后填入本地地址。完全本地推理,所有数据不离开你的 Mac。

**AI 误操作把我的文件删了怎么办?**

AI 没有权限修改或删除你 Mac 上的文件。删除类(`rm` `sudo`)和大多数修改类命令默认会被拦截。如果 AI 试图执行被拦截的命令,UI 上会显示具体原因,你可以换个问法或换条路径。

**为什么命令被拒绝?**

AI 试着执行的命令可能落在以下情况:

- 命令在白名单之外,默认拒绝(你可以换个问法)
- 命令是破坏性命令(如 `rm -rf`)
- 命令是交互式程序(如 `vim`、REPL)
- 命令会修改系统(如 `brew install`、`sudo`)

UI 上会显示具体的拒绝原因。

**为什么 macOS 弹文件夹访问权限?**

当 AI 试图读取你受 macOS 保护目录(桌面、下载、文稿)下的文件时,macOS 会弹出权限请求。这是 macOS 自身的隐私保护机制,所有类似工具都会触发。详见 [让 AI 读取文件](#让-ai-读取文件) 章节,推荐把文件放在 `~/.kaya/` 下避免。

**怎么让 AI 读取指定文件?**

把文件放进 `~/.kaya/`,然后在对话里告诉 AI 路径。详见 [让 AI 读取文件](#让-ai-读取文件)。

**怎么恢复断开的标签页?**

打开 **⚙ 设置** → **通用** 标签页的 **Shell** 小节,点击「立即重连」重连该会话对应的 PTY。

**为什么连接 LLM 时也报超时?**

首次连接 LLM 服务商最长 30 秒握手超时(网络不通、API Key 错误、Provider 限流都可能触发),UI 会显示"LLM 连接超时"横幅。检查网络与 API Key 后重试。

**怎么查看 AI 的历史操作?**

打开 **⚙ 设置** → **审计**,选择会话,查看事件流或运行评估。详见 [查看审计记录](#查看审计记录)。

**支持哪些平台?**

当前只发布 macOS 版本,需要 macOS 12 或更高,支持 Apple Silicon 和 Intel。

**脱敏等级有什么差别?**

- **aggressive**(默认):识别最多种类的敏感信息,包括身份证号、手机号、邮箱、信用卡号等
- **balanced**:识别常见 API Key、AWS 凭证、Token 等
- **strict**:只识别最明显的密钥标记

推荐保持默认的 **aggressive**。**注意**:你的输入文本始终按 aggressive 级别脱敏,不受这个设置影响。

---

### 故障排查

**终端无响应**

1. 按 **Esc** 中断 AI 当前回合
2. 打开 **⚙ 设置** → **通用** 标签页的 **Shell** 小节,点击「立即重连」重连 PTY
3. 仍无响应,尝试 **⌘W** 关闭当前标签页,再 **⌘T** 新建

**AI 回复很慢**

- 检查网络连接
- 开启思考模式会增加延迟(推理本身需要时间)
- 在设置里切换到更快的模型

**工具调用被拒绝**

UI 会显示具体拒绝原因。常见的:

- 命令是破坏性或交互式命令:换条路径
- 命令超出白名单:换个等价工具
- 试图读取 `~/.kaya/` 之外的文件:把文件放进 `~/.kaya/`

**粘贴失效**

确认焦点在正确的输入框:终端输入框的粘贴会进入终端,对话输入框的粘贴会发给 AI。

**会话丢失**

检查 `~/.kaya/conversations/` 目录是否被清理。重启应用后,旧会话应自动从磁盘恢复。如果仍然看不到,可能需要清除应用本地状态后重新开始。

**AI 一直不响应**

流式响应静默 30 秒会自动超时,UI 上会有超时提示。可以按 **Esc** 或 **⏹** 按钮手动中断。

**macOS 弹文件夹权限**

把要读取的文件放进 `~/.kaya/` 下,不要让 AI 直接访问桌面、下载、文稿等受保护目录。

**诊断日志**

遇到其他问题,打开 **⚙ 设置** → **App 日志**,点击"打开日志文件夹"查看 `~/.kaya/logs/` 下的日志文件,这些日志包含 kaya 运行时的诊断信息(不含你的对话内容)。

---

*文档版本:v1.4.0 · 最后更新:2026-08-22*

---

## English User Guide

### Welcome to kaya

kaya is a macOS desktop app that brings an AI assistant directly into the terminal you use every day. Open it and you will see two panels: a real terminal on the left, just like Terminal or iTerm2, and an AI chat panel on the right. The two share the same shell session. Whatever you see in the terminal, the AI can also see, and you can ask the AI to explain, query, or help you operate.

kaya's goal is to give you AI assistance **without leaving your real shell**: you keep your zsh aliases, your scripts, your familiar shortcuts, and the AI does not trap you in a sandbox.

> Note: the AI does not take over your entire terminal. By default it only reads your terminal output; only when you **turn on the coding-mode toggle in the top bar**, explicitly allow it on the grant card in the chat, and that tab is running a supported AI CLI (Claude Code / Kimi Code / Reasonix / OpenCode) — can the AI type into that tab on your behalf. The coding-mode toggle is the master gate for "driving": while it is off, the chat side does not have the ability to send input to terminal AIs at all (the tool is not even registered); once on, each tab still requires its own grant-card approval. The grant is bound to the current process and expires when the process exits; you can revoke it anytime via the 🤖 badge on the tab. Your regular shell prompt never accepts AI input. Likewise, when the AI helps you edit a coding agent's config file (e.g. Claude Code's settings.json), every edit lands only after you explicitly approve it on a grant card, and the grant is scoped to the current conversation.

---

### What kaya Can Do

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

### Privacy and Security Boundaries

Privacy is the **first priority** of kaya. Below is the boundary you can expect by default.

**What the AI Cannot See by Default**

- The **contents** of any file on your Mac: read commands like `cat`, `head`, `grep`, `less` are blocked by default
- Your system environment variables: `env` and `printenv` are blocked by default
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

When you send a message in the chat or let the AI call a tool, the scrubbed chat text and scrubbed tool results are sent to the **model source you selected in Settings**:

- **Built-in model (default, yunhou `/chat` proxy)**: scrubbed content is sent over your login token (JWT) to the **yunhou first-party server**, which answers on your behalf with yunhou's DeepSeek key. Before sending, the kaya client trims to the server's limits (message count / per-message bytes / total bytes — the oldest history is dropped first). Your login token (refresh_token) lives only in the macOS Keychain.
- **OpenAI compatible (your own key)**: scrubbed content goes to the LLM provider you chose (DeepSeek / OpenAI / OpenRouter / etc.), authenticated with your API key.
- **Ollama (local)**: nothing is sent (see below).

kaya does not send your content to any third party beyond the paths above. Local UI state, sessions, audit logs, and application logs all stay on your Mac.

**How Your API Key Is Used**

The API key you enter in Settings is **stored only in your local app settings**. It is sent only to the LLM provider you selected, in the authentication header of each request, for that provider's authentication. kaya does **not** send your API key to any other party. kaya does **not** promise "never uploaded" — any network request needs authentication, and authentication needs the key, that is the basic mechanism of network requests.

**Offline Inference: Ollama**

If you choose Ollama as the provider, all inference happens **entirely on your Mac**, with **no outbound requests**. This is suitable for the most privacy-sensitive scenarios.

**Five Layers of Protection, in Brief**

In short, everything from your input to the AI's output goes through:

1. **Input guard**: pasted keys and long text are scrubbed and truncated
2. **Command policy**: executable commands are restricted by an allowlist; unknown commands are denied by default
3. **Safety check**: destructive, interactive, and system-modifying commands are blocked
4. **Isolated execution**: AI tool calls run in independent subprocesses and never touch your terminal
5. **Output scrub**: tool output is scrubbed again before entering the chat

---

### Install kaya

**Primary Path: Download the DMG**

1. Visit the kaya website or the GitHub Releases page and download `kaya-x.y.z-macos-universal.dmg` (universal for Apple Silicon and Intel)
2. Double-click the DMG to mount it; a window showing `kaya.app` will appear
3. Drag `kaya.app` into the `/Applications` folder
4. Open kaya from Launchpad or `/Applications`

**First Launch**

If macOS shows "cannot be opened because Apple cannot check it for malicious software":

1. Find `/Applications/kaya.app` in Finder
2. Right-click and choose "Open"
3. Click "Open" again in the dialog

After this, you can double-click it normally.

**Advanced: Build from Source**

If you want to compile it yourself, please refer to the GitHub repository's README for step-by-step instructions. Build artifacts are in the `target/release/bundle/` directory of the repo.

> Note: pre-compiled CLI releases are currently paused. If you need the CLI, build it locally as shown above, or check the GitHub Releases history for older versions.

---

### First-time Setup

After opening kaya for the first time, do the following.

**Option A (default, no API key): built-in model + WeChat sign-in**

By default kaya uses the **built-in model** (the yunhou `/chat` proxy — the server answers with yunhou's DeepSeek key), so you don't need to apply for an API key. You only need to sign in to your yunhou account:

1. Click **⚙ Settings**, or click **Sign in with WeChat** in the chat window (the account icon / sign-in button in the top bar)
2. Your system browser opens a WeChat scan page → scan → you're redirected back to kaya (the token is written to the macOS Keychain)
3. Return to the chat window and ask away. Subscription / trial status is under **⚙ Settings → Account**

**Option B (your own API key): OpenAI compatible**

If you want to use your own LLM key:

1. Open **⚙ Settings**, and in the **LLM** section of the **General** tab, switch **Model source** to **OpenAI compatible**
2. Select your **Provider**:
   - **DeepSeek**: commonly used in mainland China; default model `deepseek-v4-flash`
   - **OpenAI**: international, requires overseas access
   - **OpenRouter**: aggregates many models; switch models under one key
   - **Ollama**: local inference; requires `ollama serve` to be running
   - **Custom (OpenAI compatible)**: bring any OpenAI-compatible endpoint with its Base URL and model
3. Enter your **API Key**, obtained from your provider's website
4. Check the **Base URL**: it is auto-filled when you pick a preset; usually you do not need to change it
5. Select a **Model**: the default works for most tasks
6. Optionally turn on **Thinking Mode**: lets the AI show its reasoning before answering
7. Click Save, then return to the chat

Once configured, kaya remembers your settings. You do not need to reconfigure next time.

---

### Your First Conversation

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

### Slash Commands in Chat

Type **`/`** in the chat input to summon the command menu (pick with the keyboard or mouse to autocomplete); `/help` lists all commands anytime. The built-in commands:

- **`/agents`**: lists the currently running companion agents (kimi / claude / opencode / reasonix, etc.). Click a card to bind it as this session's forward target — the binding is per-session and cleared on a new conversation
- **`/new`**: start a new conversation (same entry point as the New button in the top bar)
- **`/sessions`**: lists past sessions from the audit trail; click a card to view its read-only history
- **`/help`**: lists every command with a description

Input that starts with `/` but isn't a built-in command (such as `/auto`) does not error out: if the current session has a bound agent, kaya writes the original text into that agent's terminal input line and presses Enter for you. Before writing, it fails closed on several checks (the pane is still alive, the foreground process is still the bound agent, and it is not waiting at a menu); if any check fails, nothing is written. These forwards **do not consume an LLM turn**; each forward appears as a card in the message stream and is recorded in the audit log. When you re-bind across tabs, kaya injects a one-time context note to the newly bound agent so it knows what came before.

---

### Task List

kaya ships a lightweight task list to help you and the AI track what needs doing:

- The todo toggle in the top bar opens the **Task panel**, where you can view, check off, and manage current tasks
- You can also manage it in natural language right in the chat: ask the AI to **extract** tasks from the conversation, or to **view / add / edit / delete / clear** the list
- On the AI side there is a single **read-only** tool (viewing the task list) to align on context — it cannot modify the list for you

---

### Terminal and Tabs

**Basic Terminal Operations**

The left panel is your real zsh. Common actions:

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

Each tab has its own working directory, command history, and scrollback. When a tab is closed, its command history is automatically merged into your global `~/.zsh_history`.

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

Turn on the coding-mode toggle in the top bar and kaya proactively watches the coding agents running in your tabs (Claude Code / Kimi Code / Reasonix / OpenCode, etc.) — no need to watch the terminal yourself:

- When an agent **pops up a confirmation and waits for your decision**, kaya automatically starts a chat turn, relays what decision is needed, and asks for your instructions
- When an agent **finishes its task (the screen goes idle)**, kaya proactively reads the result, summarizes the progress, and asks about next steps
- The watch is **read-only**: kaya only reports and asks — it never clicks a confirmation for you; the decision is always yours
- This toggle is also the **master gate for "driving"**: only while it is on can the chat side send instructions to a coding agent's tab (each tab still needs its own grant-card approval); turn it off and that capability is withdrawn **from the next turn onward** (the tool is no longer registered; a turn already in progress finishes with the state it started with). What remains active with the toggle off is just the passive notification cards and tab badges
- The confirmation summary carried into an auto turn is secret-scrubbed; each tab has a 60-second cooldown, and kaya never interrupts an ongoing chat turn (it follows up after the turn finishes)
- Note: coding-mode auto turns are sent to your selected model over the same path as your manual questions; with the toggle off, nothing is triggered at all

**Agents Panel**

Open **Agents** from the app menu (or click the Agents icon on the right side of the top bar). Installing, updating, and model configuration can also be done by asking kaya in chat (e.g. "install Claude Code for me") — both paths run the same engine:

- **Background install / update** of coding agents: before installing, kaya automatically picks one available method (npm > brew > the official script), with live progress shown on the card; if the install fails there is no automatic fallback to another method — use "Install in terminal" instead
  - If the npm method hits a network-class failure, kaya retries **once** via the npmmirror registry; brew and the official script are not retried
  - Installs have no overall timeout: if nothing seems to move, check the output tail (last 50 lines) on the panel card to tell slow from stuck
  - Uninstall is supported for npm/brew-managed installs (with an ownership check — an install not managed by that package manager is never touched) and removes only the program itself, keeping config and history; script-installed agents must be removed manually per the official docs
- **Model provider library** (cc-switch style): keep multiple provider configurations (base URL / model / API key) per agent; clicking **Use** writes the entry into that agent's own config file (a `.bak` backup is kept)
  - When kaya edits a config from chat, a grant card asks for your confirmation before anything hits disk; secret values are never echoed back into the chat
  - To roll back, manually restore the `.bak` backup written before the change (e.g. `~/.claude/settings.json.bak`)
  - Note: provider API keys are stored in **plaintext** at `~/.kaya/agent-providers.json` (same as cc-switch) — do not share this file with others
- Over IM channels (Feishu / WeCom): only config read/edit is available (grant cards included); status checks, install, and update are not available there

---

### Let the AI Read a File

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

### Web Search

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

### Use kaya via Feishu / WeCom

Since v2.1.0, kaya can bridge IM messages to your Mac: message the bot in Feishu or WeCom (Enterprise WeChat) and your local kaya LLM handles it and **streams a reply** — no need to have the kaya window open.

**How to enable**

1. Open **⚙ Settings** → the **IM Channels** tab (also reachable directly from the system menu: kaya → **IM Channels…**) and pick a platform — **the WeCom entry is currently greyed out as "(coming soon)"**; only Feishu is open for setup in this version (existing WeCom configs are preserved and will be re-enabled in a later version)
2. **Recommended: click "Connect via Feishu QR"** (Feishu only): scan the QR code with the Feishu mobile app and confirm — kaya automatically creates the app and fills in the App ID plus the creator's open_id in the allowlist; the App Secret goes straight into the **macOS Keychain**, never to disk, never echoed back
   - Manual configuration (fallback): create an app yourself on the [Feishu Open Platform](https://open.feishu.cn) or the WeCom admin console, then enter the Feishu App ID + App Secret, or the WeCom BotID + Secret
3. Configure the **admission allowlist**: one user ID per line (Feishu `open_id`, starting with `ou_`; WeCom member userid); QR provisioning locks it to the creator automatically
4. Click **Start channel**; once the status shows "running", message the bot from your IM. **If a permission error appears**, enable the im:message scopes for this app and publish a version on the Feishu Open Platform

**Key points**

- **Direct long connection to the platform**: Feishu official WebSocket, WeCom official openws channel — **no public IP or callback server needed**
- **Allowlist denies strangers by default**: messages from users not on the list are dropped before ever reaching the LLM; an empty list prevents startup
- **Same privacy pipeline**: IM messages go through the same five-gate pipeline as desktop chat; IM turns use the same full tool table as the desktop (terminal read / task list included), while `terminal_write` is still governed by the coding-mode master switch — that switch is an atomic flag shared between desktop and channel; toggling it on the desktop is pushed to the channel immediately via `channel_set_coding_mode`, taking effect from the next IM turn
- **Desktop → IM mirroring (opt-in, off by default)**: once enabled, the desktop chat stream is pushed in real time to your chosen IM chat, and your own inputs are echoed as "你:…"-prefixed text; IM replies also appear in the desktop chat panel. Starting a new desktop session migrates the mirror automatically and pushes a "new session started" notice. The toggle lives in two places — the "Mirror desktop chat to Feishu" checkbox in Settings and the mirror icon next to coding in the top bar; both are the same switch and take effect immediately
- **Feishu authorization cards**: when an IM turn needs terminal_write / agent-config approval, Feishu shows an interactive card with Approve / Deny buttons; decisions share the same grant store as desktop approval cards (mutually recognized); the operator must be on the admission allowlist. Once approved, kaya **automatically runs a follow-up turn** asking the AI to retry the paused operation — no need to wake it up with another message
- **Proactive agent-status notifications**: with mirroring on, the coding agent's waiting-for-input / idle-done / online / exited transitions are pushed to the mirrored chat (Chinese or English per the channel locale, with per-event cooldown to avoid flooding)
- Sessions and audit land in `~/.kaya/` as usual; the LLM uses the model source configured in General
- **Honest caveat (WeCom)**: WeCom mirror echoes and proactive notifications depend on an active callback registration and may be silently dropped in pure-desktop scenarios (no IM turn in flight); only Feishu was verified on a real device in this round

See the "IM channel entry" and "IM channels & mirroring" sections of [docs/PRIVACY.md](./PRIVACY.md) for details.

---

### Thinking Mode

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

Open **⚙ Settings** → **LLM** → find **Thinking Mode**, and turn it on. Some chat sessions can also control it per turn if the session is enabled.

**Switching the Reasoning Model**

kaya's Settings has a **Reasoning Model** field.

> **Important**: For **chat requests**, the **Reasoning Model** field is currently **display-only** — it does not switch the actual model ID in the AI's request. The field is currently used only by the **AI judge** in audit evaluations (to pick the judge model).

To switch which model the AI uses, change the **Model** field in Settings. When Thinking Mode is enabled, kaya automatically adapts reasoning parameters per the selected provider's rules (for example, DeepSeek injects a `reasoning_effort` parameter) — **you do not** need to switch to a separate "reasoning-only model".

If you see the Reasoning Model field, treat it as a **reserved** field for now; its value does not affect chat requests.

---

### Sessions and History

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

### Audit Records

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

### Settings Reference

Open **⚙ Settings**. The tabs are: **General / IM Channels / Audit / App Logs / Shortcuts / Account** (dev mode also has Debug). **LLM, Web Search, Shell, Language, Scrubbing** are all sections inside the **General** tab, not standalone tabs.

**General → LLM section**

| Item | Description |
|---|---|
| Model source | **Built-in model** (yunhou proxy, no key, default) / **OpenAI compatible** (your own key) |
| Provider | LLM provider (OpenAI compatible mode only) |
| API Key | Provider key, stored only on this machine (OpenAI compatible mode only) |
| Base URL | API endpoint; auto-filled when you pick a preset (OpenAI compatible mode only) |
| Model | Main chat model (OpenAI compatible mode only) |
| Reasoning Model | Display field (currently used only by the AI judge in audit evaluations — see Thinking Mode) |
| Thinking Mode | Enable thinking mode |
| Scrub Level | Scrubbing strength (aggressive / balanced / strict); default aggressive |

> Note: **Max Context Messages** controls the Rust agent loop's `conversation::trim_history` (keeps the system prompt + the most recent N messages). Wired in 2026-08-05 review C1 (`agent.rs::reload_trimmed`). The persistence cap is 200 messages per conversation (`conversation.rs::MAX_MESSAGES_PER_CONVERSATION`).

**General → other sections**

| Item | Description |
|---|---|
| Language | UI language |
| Shell | Shell path used by the PTY |
| Preserve terminal scrollback | Whether to keep each tab's scrollback after restart |
| Terminal font size | Terminal display size |
| Web Search | Enter the Tavily API key and enable web search |

**Account**

View sign-in status, subscription / trial, refresh subscription, sign out; the Advanced section lets you paste a login token manually as a fallback.

**Audit**

View past sessions and run evaluations.

**App Logs**

View diagnostic logs and **manually** prune logs older than 7 days.

**About**

About is not a Settings tab. Open the standalone About dialog from the **system menu** (the kaya menu in the macOS menu bar → About kaya) to view the version, build, and license; **Check for Updates…** lives in the system menu too. The system menu also provides **IM Channels…** and **Keyboard Shortcuts…** shortcuts that jump straight to the corresponding Settings tabs.

---

### Keyboard Shortcuts

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

### FAQ

**Can the AI read my files?**

By default, no. Commands like `cat`, `head`, and `grep` are blocked by default. If you need the AI to read a file, place it under `~/.kaya/` and tell the AI the path. See [Let the AI Read a File](#let-the-ai-read-a-file).

**Will pasted keys be sent out?**

No. Keys you input are automatically scrubbed inside kaya (PEM private keys as whole blocks, API keys, ID numbers, phone numbers, emails, credit card numbers, and other common patterns). Only the scrubbed text enters the chat and the logs. Replaced parts are marked "scrubbed" in the UI.

**Is my API key safe?**

Your API key is stored only in your local app settings. It is sent only to the LLM provider you selected, in the authentication header of each request, for that provider's authentication. It is **not** sent to any other third party. kaya does **not** promise "never uploaded" — any network request needs authentication, and authentication needs the key; that is the basic mechanism of network requests.

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

Only macOS is currently released. Requires macOS 12 or later, supporting both Apple Silicon and Intel.

**What's the difference between scrub levels?**

- **aggressive** (default): recognizes the most categories of sensitive information, including ID numbers, phone numbers, emails, and credit card numbers
- **balanced**: recognizes common API keys, AWS credentials, tokens
- **strict**: recognizes only the most obvious key markers

Recommended: keep the default **aggressive**. **Note**: your input text is always scrubbed at the aggressive level regardless of this setting.

---

### Troubleshooting

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

*Document version: v1.3.0 · Last updated: 2026-08-18*
