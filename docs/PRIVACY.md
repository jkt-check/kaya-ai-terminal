# kaya · Privacy First 设计

> **第一优先级是用户隐私,绝不把隐私信息报漏给 LLM。**
>
> 这是产品决策的起点,不是文档末尾的"安全提示"。所有
> 设计在写第一行代码前,先问一句:**"这条数据出去之后能不能撤回?"** 不能
> 撤回,就在出门前 redact 掉。

---

## 1. 一句话承诺

> kaya 运行在**你自己的 Mac** 上,LLM 只看得到你**主动给
> 它**的内容。**默认它什么文件都看不到**;你往 `~/.kaya/`
> 下放了什么、你说的话里有什么 secret,被 5 道关卡过完之后,才进
> LLM 的 context。

这是结构性的,不是配置项 — 改不了。

---

## 2. 数据流:什么出机器、什么留本地

```
你的 Mac                                                          LLM endpoint
┌──────────────────────────────────────────────┐                ┌──────────────┐
│                                              │                │              │
│  ┌──────────┐   run_shell (独立子进程)        │                │              │
│  │ 真 zsh   │ ──────────────────────► [policy]│                │              │
│  │ (用户)   │                    │           │                │              │
│  └──────────┘                    │           │                │              │
│        ▲                         ▼           │                │              │
│        │  PTY 输出                [safety]   │                │              │
│        │  (LLM 是观察者)            │           │                │              │
│        │                           ▼           │                │              │
│        │                       [exec]          │                │              │
│        │                           │           │                │              │
│        │                           ▼           │                │              │
│        │                       [scrub]          │                │              │
│        │                           │           │                │              │
│  ┌──────────┐                      │           │   已脱敏的      │              │
│  │ Chat UI  │ ◄──── LLM 响应 ─────┘           │   请求体        │              │
│  │ (前端)   │ ────────────── invoke("llm_chat") ───────────────► │              │
│  └──────────┘                                  │                │              │
│      │                                         │                │              │
│      ▼                                         │                │              │
│  localStorage (UI 状态,reasoning 不存)         │                │              │
│  ~/.kaya/                            │                │              │
│    ├─ conversations/c_*.json (LLM 上下文)        │                │              │
│    ├─ audit/s_*/events-*.jsonl (审计)            │                │              │
│    └─ debug/s_*/turn-N.json (可选, dev-only)     │                │              │
└──────────────────────────────────────────────┘
```

**只有三样东西**会离开你的机器:scrub 后的用户输入、scrub 后的工具
输出、scrub 后的 LLM 请求体。**API Key 本身**只发到 LLM 的鉴权 header,
不进 body;**Prompt 内容**是 scrubbing 后的内容,不是原始输入。

**全部状态都在本机**:UI 状态在 `localStorage`,LLM 上下文 / 审计 / 调试
在 `~/.kaya/`。状态存储无中心服务器,无遥测,无云同步。

**默认模型来源(round-138, 2026-08-04)的数据出口**:kaya 出厂默认走
**内置模型**(yunhou `/chat` 代理)—— 脱敏后的对话内容经你的登录令牌
(JWT)发往 **yunhou 第一方服务器**,由服务端持 DeepSeek key 代答。客户端
发送前按服务端限额裁剪(消息条数 / 每条字节 / 总字节上限,超限丢最早历史
—— 发送前丢内容,本身是隐私相关行为)。登录令牌(refresh_token)只存在
macOS 钥匙串。改用 **Ollama**(本地推理)可做到零外发;改用 **OpenAI 兼容**
则内容发往你自选的服务商。本机状态存储仍无中心服务器 —— 但默认数据
**出口**经 yunhou 服务器,这是 round-138 起的变化,隐私定位需明确知情。

---

## 3. 5 道关卡:scrub 是兜底,policy + safety 是主防

设计哲学:**不要相信 scrub 能拦住一切**。**结构上** LLM 默认拿不到
文件内容(没有 cat、没有 less、没有 base64),**只有**用户主动放到
trusted root 下的内容才放行。Scrubber 是 regex,regex 必然有漏判 —
它的工作是"减少残留",不是"防泄漏"。

```
   用户输入 (你打字 / 粘贴的内容)
       │
       ▼
   ┌──────────────────────────────────────────┐
   │  第零关 · 用户输入 guard                  │
   │  位置: 前端 runAgent 入口 + Rust llm_chat │
   │  动作: scrub-first → truncate 32 KiB      │
   │  失败模式: scrub regex 漏判                │
   │  兜底: 「已脱敏」badge 提示                 │
   └──────────────────────────────────────────┘
       │
       ▼  (scrubbed text 已是 redact 后内容)
   ┌──────────────────────────────────────────┐
   │  第一关 · Policy gate (能力策略)           │
   │  位置: Rust run_shell stage 1            │
   │  动作: 黑名单/白名单 + 管道分段 + trusted  │
   │         read (cat ~/.kaya/*   │
   │         / head / tail / grep / awk /     │
   │         sed -n 等)+ trusted download     │
   │         (curl -o ~/.kaya/     │
   │         <安全后缀> URL)                   │
   │         + redirect 门控(> / >> 仅当目标  │
   │         在 ~/.kaya/ 内且扩展名安全时放行, │
   │         round-68)                        │
   │  失败模式: 新加的"危险命令"没被覆盖         │
   │  兜底: 默认拒绝,LLM 收到结构化错误          │
   └──────────────────────────────────────────┘
       │
       ▼
   ┌──────────────────────────────────────────┐
   │  第二关 · Safety gate (危险命令检测)       │
   │  位置: Rust run_shell stage 2            │
   │  动作: DANGEROUS/REPL/OPERATION 正则      │
   │  失败模式: heredoc / $(...) / `...` 嵌套   │
   │  兜底: 递归扫描 depth=3 + base 正则        │
   └──────────────────────────────────────────┘
       │
       ▼  (通过的命令)
   ┌──────────────────────────────────────────┐
   │  第三关 · Execute 隔离                      │
   │  位置: Rust run_shell stage 3            │
   │  动作: zsh -f -c <cmd> (不读 .zshrc)       │
   │  设计: 与用户 PTY 完全独立                  │
   │  限制: 默认 8s 超时(用户可配,du/find/ │
   │         grep 等慢扫描命令下限 30s)+     │
   │         100KB 输出上限                  │
   └──────────────────────────────────────────┘
       │
       ▼  (执行产出)
   ┌──────────────────────────────────────────┐
   │  第四关 · Output scrub (凭据脱敏)          │
   │  位置: Rust run_shell stage 4            │
   │  动作: ~38 类 secret/PII 正则,整块 PEM    │
   │  失败模式: base64 编码过的 secret         │
   │  兜底: 用户主动开 trusted read 时仍会 scrub │
   └──────────────────────────────────────────┘
       │
       ▼  (scrubbed output)
   进 LLM context
```

**用户在 terminal 里敲的命令不经过任何门** — 那是你的 shell,你的责任。

---

## 4. 第零关:用户输入 Guard

跑在**两处** — 前端 `runAgent` 入口 + Rust `llm_chat` 入口。两次跑同一
个 pipeline,**任何一头失败都不让 raw text 出门**。

### 4.1 顺序关键:先 scrub,再 truncate

如果先 truncate 再 scrub,secret 可能**正好骑在截断边界**上,被切一
半泄漏给 LLM — 截断后的前半段就是真实 secret 的前缀。scrub-first
保证:任何被 redact 的 span 要么完整保留 (`[REDACTED:private-key]`),
要么完全消失,不会切一半。

实现:

- 前端: `src/lib/userInputGuard.ts::applyUserInputGuard`
- Rust: `crates/kaya-core/src/llm/user_input_guard.rs::apply_user_input_guard`
- CLI: `crates/kaya-cli/src/input_guard.rs::guard`

### 4.2 32 KiB 硬上限

`USER_INPUT_MAX_BYTES = 32768`(v1.2.3: 4096 → 65536 → 32768),前端和 Rust 各自
一个常量,锁在 cargo tripwire + vitest test 里。截断按 UTF-8 码点二分,
emoji / surrogate pair 不切一半。

### 4.3 全 PEM 整块 redact(2026-06-19 事故)

> 事故复盘:一个用户粘贴了完整 RSA 私钥,旧 regex 只匹配
> `-----BEGIN ... PRIVATE KEY-----` 标记行,**整段 base64 body 直接
> 走 LLM**,在 chat UI 上明文显示。

修复后,`-----BEGIN ... PRIVATE KEY-----` + body + `-----END ... PRIVATE
KEY-----` 整块匹配(非贪婪 body),整段密钥作为一整块
`[REDACTED:private-key]`。支持 PKCS#8 unencrypted / RSA / OPENSSH /
DSA / EC / ENCRYPTED / PGP 7 种类型变体。Rust 端 regex 不支持
backreference,所以枚举了 7 个 BEGIN/END 对作为 alternation。Type
alternation 是**可选**的 — 抓 PKCS#8 (`-----BEGIN PRIVATE KEY-----`)
这种没 type 前缀的。

Rust 和 TS 端共享同一份正则的访问入口(`crates/kaya-core/src/llm/
scrub.rs::ssh_key_regex()`),避免两边漂移。

### 4.4 `originalBytes` 反映原始输入

UI 提示"原 5.0 KiB → 4.0 KiB"必须用**原始**输入字节数,不能用
post-scrub 字节数 — 一次 scrub 可能把 5 KiB 的粘贴缩到 1 KiB
(PEM body 占了 4 KiB),用户看到 "原 1 KiB" 会以为"我就打了 1 KiB"。
回归测试 `userInputGuard.test.ts::originalBytes reflects RAW input bytes`
锁这条不变量。

### 4.5 命中时 UI 显式提示

用户消息命中 redaction 时,UI 渲染「已脱敏」badge;截断时显示"已截断:
原 N KiB → M KiB"。**用户清楚知道发生了什么**,而不是"系统默默改了我的话"。

### 4.6 IM channel 入口(2026-08-22,v2.1.0)

飞书 / 企业微信是 chat 输入框之外的**第二个用户输入入口**:允许名单里的
用户在 IM 里给机器人发消息,由本机 kaya 的 LLM 处理并流式回复。入口换了,
防线不换 —— 每条 inbound 消息走的仍是**同一条五道关卡 pipeline**:

- **第零关照跑**:消息在进 agent loop 前过同一份 `user_input_guard`
  (scrub-first → truncate 32 KiB);工具调用照常经 policy → safety →
  隔离执行(`zsh -f -c` 独立子进程)→ output scrub。对话内容只发往你在
  「通用」页选定的 LLM provider 路径,kaya 本身不经任何第三方中转。
- **准入 allowlist(默认拒绝陌生人)**:按平台用户标识准入 —— 飞书填
  open_id(`ou_` 开头),企业微信填成员 userid。不在名单里的用户消息
  **直接丢弃**,不进 LLM、不产生回合、不留 conversation。名单为空时
  渠道无法启动 —— 没有准入限制等于任何人都能让 LLM 在本机执行命令。
- **coding 能力受同一套总闸 + 授权闸约束**(2026-08-22 镜像对齐):
  IM turn 的工具表与桌面全集对齐,但 `terminal_write` 仅当 coding
  模式总闸快照开启时才注册,且执行前仍逐 tab 检查授权表 + 每分钟写
  预算(与桌面同一检查点);`agent_config` 的 edit 仍走共享授权闸,
  未授权只发授权请求、不写盘。飞书侧授权决议经**交互卡片**做出
  (2026-08-22 P2,见 §4.7);企微侧本轮无卡片,决议仍在桌面端做出,
  IM 侧无法自我授权。
- **凭证只存 macOS 钥匙串**:飞书 App Secret / 企微 Secret 在保存时写入
  Keychain,不落盘文件、UI 不回读显示,也绝不随对话内容发往任何 LLM provider。
- **长连接直连平台**:飞书走官方 WebSocket 长连接(open-lark),企微走
  `wss://openws`,**不需要公网 IP 或回调服务器**,也没有第三方中转;
  会话映射落 `~/.kaya/channels/sessions.json`,镜像目标 chat 落
  `~/.kaya/channels/mirror.json`(2026-09-01,镜像绑定跨重启恢复),IM
  会话复用既有的
  conversation JSONL 落盘(`~/.kaya/conversations`,见 §8)。**如实说明**:
  channel 入口暂未接入审计 JSONL —— audit trail 里没有 IM 收发行,
  该缺口计划后续 round 补齐;五道关卡本身不依赖审计日志,在 IM 管线内
  照常逐关执行。

### 4.7 IM 渠道与镜像(2026-08-22)

镜像是 §4.6 之上的**可选增强**,把「对话内容出本机」的边界从 LLM
provider 一条路径扩到两条,因此全部默认关、显式 opt-in:

- **镜像 = 全项目唯一的内容出本机新增路径,默认关**。开启前,出本机的
  对话内容只发往生效的 LLM provider;开启「桌面 → IM 镜像」后,桌面
  chat 的消息流(含你的输入回显「你:…」)会实时推送到你指定的 IM 聊天。
  开关在 Settings → IM 渠道,运行中可即时切换(`channel_set_mirror`),
  关镜像不断绑定 —— 重开时原 chat 继续镜像,不重复通知。
- **双向镜像共用同一条出站管线**:桌面 → IM 方向经 FanoutSink 复用
  ChannelEventSink(与 IM turn 同一渲染 / 节流 / 出站);IM → 桌面方向
  把 IM turn 的事件扇出到 WebView(与桌面 llm_chat 同一个 sink 实现)。
  内容在源回合已过 output scrub 第 4 关(桌面回合按用户设置档位,
  IM 回合固定 `ScrubLevel::Strict`),镜像出站管线本身不再重复 scrub。
- **飞书授权卡片等效桌面授权卡**(2026-08-22 P2):terminal_write /
  agent_config edit 的授权请求在飞书渲染成带「批准 / 拒绝」按钮的交互
  卡片,决议写入**与桌面共享的同一份 grant store** —— 飞书批的桌面认,
  桌面批的飞书认,不存在第二套授权状态。安全边界:操作者必须逐字在
  准入 allowlist(与入站消息同一道闸,陌生人回调静默丢弃);按钮指纹
  由本机组装、回调解析失败一律 fail closed;决议留痕 app_log,操作者
  只记长度不记内容。
- **agent 状态主动通知**(2026-08-22 P3):镜像开启时,绑定 tab 里的
  编程 agent 状态变化(等输入 / 空闲完成 / 上线 / 退出)主动推送到镜像
  聊天,按渠道语言快照出 zh/en。通知里的画面摘要(excerpt)与桌面
  「待决策条」用的是**同一段已过 scrub 的文本**(watcher emit 前过
  scrub_text 同一管道),不是新的内容出口;per (tab, kind) 冷却
  (60s / 300s / 30s)防通知风暴;推送全部失败静默,绝不影响 watcher
  与 IM 主链路。
- **IM 工具表已与桌面全集对齐**(2026-08-22 镜像对齐):terminal_read /
  terminal_write / agent_config / task_list 与桌面共享同一份 store
  (终端历史 / writer / grant / abort 表 / 任务清单快照)。`terminal_write`
  仍由 coding 模式总闸照管 —— **总闸是桌面与渠道共享的原子标志,桌面开关
  扳动经 `channel_set_coding_mode` 即时写入渠道侧,下一 IM turn 生效**;
  写入本身仍受逐 tab 驾驶授权约束。
- **如实说明(企微限制)**:企业微信的镜像回显(UserEcho)与主动通知
  依赖活跃的回调登记(纯桌面场景没有 IM turn 在途时,出站消息可能
  **静默丢失**);本轮仅飞书做了真机验证,企微镜像能力以代码对齐为准、
  未经真机验收。

---

## 5. 第一关 + 第二关:Policy + Safety Gate

LLM 调 `run_shell` 走 Rust 端 4 阶段 pipeline 的前两阶段:

### 5.1 Policy(能力策略,3 语义表 + 7 个 deny 类别)

源码: `crates/kaya-core/src/llm/policy.rs`(280 个 cargo test,2026-08-31 重测)。round-67(2026-07-05)起,能力策略**不再是黑/白名单二元结构**,而是 **3 个语义表(METADATA / STREAM_TRANSFORMERS / FILE_READERS)+ 永久 blacklist + allowlist-miss 兜底**,deny reason 带结构化前缀 `[stage:subcategory]`(详见 §5.1.1)。系统 prompt(`policy_prompt::render_allowed_section_*`)与 run_shell 工具描述(`tools/run_shell.rs::rules_prose`)从同一份数据源生成,单一真相,无法漂移。

- **trusted_download 子阶段(round-57, 2026-06-28)**:LLM 可以用
  `curl -o ~/.kaya/<file> URL` 把外部上下文下载到 trusted
  root。`-o` / `--output` / shell-redirect `>` 解析 write target,后缀
  过安全检查(.sh / .py / .exe / .dmg / 等被拒,见
  `BLOCKED_DOWNLOAD_EXTS`);URL 必须 http(s) 且不指向
  loopback / link-local / cloud-metadata(SSRF)。**v2.1.10 review
  M-sec-1 收紧**:trusted download 拒收 `-L` / `--location` ——
  跟随 302 重定向会绕过首跳 URL 校验(重定向目标可指向任意
  主机);想跟重定向就再发起一跳,每跳都过 URL 校验。**新隐私面**:
  下载的字节本身不直接进
  LLM —— 只有 LLM 后续 `cat` 才会通过 trusted_read 读回;文件落在
  用户可见路径,可由用户随时检查 / 删除。详见
  [`docs/superpowers/specs/2026-06-28-trusted-download-design.md`](./superpowers/specs/2026-06-28-trusted-download-design.md)。

  > Windows:PowerShell 里 `curl` 是 Invoke-WebRequest 别名,trusted download 教/用 `curl.exe` 显式形态(`table_base_key` 剥 `.exe` 归一后同样命中 `curl` 白名单条目,2026-08-25 W2)。

- **3 个语义表**(round-67,见 §5.1.1):**METADATA**(`ls` / `ps` / `df` / `uname` 等纯元数据,永远放)、**STREAM_TRANSFORMERS**(`head` / `sort` / `grep` / `awk` 等,非首位永远放,首位无文件参数或文件在 `~/.kaya/` 放)、**FILE_READERS**(`cat` / `xxd` / `strings` / `base64` 等,仅文件在 `~/.kaya/` 放)。`cat` / `xxd` / `strings` / `base64` 在 `~/.kaya/` 下**条件放行**(trusted_read),不是一律黑名单。**v2.1.10 review C1 收紧**:`source` 移出 FILE_READERS —— `source <file>` 是**执行**文件不是读文件,留在表里会与 trusted download 组合成无确认 RCE 链;现一律落 allowlist-miss 拒(`. file` 形态仍由 dot-source blacklist 拦)。
- **永久 blacklist**(永不放,deny 前缀 `[policy:blacklist]`):shell escape / `eval` / `exec` / `xargs` / `curl`(非 trusted_download)/ `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` 等。**归属注意**:`rm` / `sudo` 由 safety 层 DANGEROUS 拦(`[safety:dangerous]`),`osascript` 由 safety 层 OPERATION 拦(`[safety:operation]`),`tee` 在 round-67 移出 STREAM_TRANSFORMERS 后落入 `[policy:allowlist-miss]` —— 这 4 个都不在 policy 层永久黑名单里。
- **管道分段识别**:`du | sort | head` 中,`sort` / `head` 在非首位时是"纯字节流转换器",**不**触发内容读取黑名单。**2026-08-20 review C2 收紧**:管道消费者带**显式文件路径参数**时(`ls | head -100 ~/.ssh/id_rsa`)不再是纯 stdin 变换,降级为 trusted-read 语义 —— 路径必须进 `~/.kaya/`,否则 `[policy:trusted-read-miss]` 拒绝;仅无路径参数的形式保留无条件跳过。**C2 误杀回归修复(同日复查)**:`tr` 不接受文件操作数(GNU/BSD 都报 extra operand,SET 是字符集不是路径),永远按 stdin-only 放行;`cut` / `sort` / `uniq` 的 value-taking flag 空格分隔形式(`cut -d ':'` / `sort -t ':'` / `uniq -f 2`)的值不再被误当路径(两侧 VALUE_FLAGS 同步补表)。**2026-08-20 review C1 收紧**:`--name=value` 内联(`sed --expression=p file`)与短选项粘连(`-eSCRIPT` / `-n5`)按 value-taking flag 识别,不再把 `=` 内联值当普通 flag 跳过 —— 否则紧跟其后的文件操作数会被误读为 script/pattern,路径集抽空后走"stdin-only 放行"分支,任意文件内容可读。两侧(Rust `policy.rs` ↔ TS `runShellPolicy.ts`)同步修复 + 回归测试。
- **Trusted Read 子阶段**(2026-06-17,见 `2026-06-17-trusted-read-root-
  design.md`):如果 path args 归一化到 `~/.kaya/` 下,放行
  `cat` / `head` / `tail` / `grep` / `awk` / `bat` / `view` / `sed -n`
  (写操作按类型分流:`sed -i` / `tee` / `mv` / `rm` **仍全拦**;`>` / `>>`
  重定向自 round-68(2026-08-05)起,当目标归一化到 `~/.kaya/` 内且扩展名
  安全(非空、不在 BLOCKED_DOWNLOAD_EXTS)时放行,`/dev/null` 例外;
  `<>` 按写门控,`<` 输入重定向按 trusted-read 规则门控)
- **默认拒绝(allowlist-miss)**:命令不在 3 个白名单表 = 拒,LLM 收到结构化错误,允许它换种方式重试

- **v2.1.10 review 安全收紧(H1 / H2 / M-sec-2 / M-sec-3,两侧镜像同步)**:
  - **sed script 命令(H1)**:在既有 sed `w`(写)防护同形状上,新增 sed
    script body 内 `r` / `R`(把任意文件读进输出流,如
    `sed -n '1r /etc/passwd' ~/.kaya/x.txt`)与 GNU sed `e`(执行 shell
    命令)的无条件拒绝(`[policy:blacklist]`,trusted_root 无例外);
    管道位(`ps aux | sed -n '1r ...'`)借 pipeline-consumer skip 的绕过
    一并堵上(Layer 1 对 sed 的 w/r/R/e 不再跳过)。正常 `sed -n '1,5p'`
    读不受影响。
  - **Windows env 引用(H2)**:Windows flavor 的 METADATA 短路(`ls` /
    `Get-ChildItem` / `Write-Output` 等 base 命中即放行)此前不看参数,
    `ls env:` / `Get-ChildItem env:` / `Write-Output $env:OPENAI_API_KEY`
    会把环境变量(含 API key)直出(env 黑名单在 Layer 7,够不到)。
    现在 Layer 2 内对 Windows flavor 做参数级扫描:token 含 `env:`
    PSDrive 引用或 `$env:` 展开即拒(`[policy:blacklist]`;Unix 行为
    逐字节不变;`$env:USERPROFILE` 拼 trusted 路径的 METADATA 用法是
    刻意接受的误杀,改写 `%USERPROFILE%` 形态即可)。
  - **无路径递归搜索(M-sec-2)**:`grep -r <pat>` / `-R` /
    `--recursive`(无文件操作数)按 GNU 语义递归读**当前工作目录**,
    裸 `rg` / `ag` / `ack` 默认递归 —— 此前空路径分支当 stdin-only
    放行。现在补隐式 `.` 的 trusted-root 判定:cwd 在 `~/.kaya/` 外则
    拒(`[policy:trusted-read-miss]`);管道位 `ls | grep -r x` 同样
    处理(grep -r 忽略 stdin 读 cwd),管道位 `ls | rg x` 读管道不受影响。
  - **METADATA 表的内容/大小 oracle(M-sec-3)**:`wc` / `md5` /
    `shasum` / `sha256sum` 带文件路径参数时补 trusted-root 判定 ——
    `wc -c ~/.ssh/id_rsa`(大小 oracle)/ `md5 <file>`(内容 hash
    oracle)在 root 外即拒(`[policy:trusted-read-miss]`);无路径参数
    的 stdin / 管道用法(`du | wc -l`、`md5 -s hello`)与 `ls` / `stat`
    等真·元数据命令不受影响。

- **v2.1.10 review R2 安全收紧(C-R2-1 / H-R2-1 / H-R2-2 / M-R2-1 / M-R2-2 /
  L-R2-3,两侧镜像同步)**:
  - **`command` 前缀执行面(C-R2-1,Critical)**:`command` 在 METADATA
    是为 `command -v <name>` 纯查询,但 `command <cmd> [args…]` 是通用
    执行前缀,Layer 2 短路不查参数时 `command cat /etc/passwd` /
    `command sh -c 'id'` / `command env` / `command xargs ls` 全部放行
    (等价任意命令执行,还绕过 `\bxargs\b` 这类命令名黑名单)。现在
    Layer 2 对 base == `command` 加特例:只放行 `command -v/-V <name>`
    纯查询,其余形态一律 `[policy:blacklist]` 拒(fail-closed,不剥
    前缀重跑分类)。
  - **sed w/r/R/e 连写 / 无空格 / 相对路径缺口(H-R2-1)**:H1 的
    w/r/R/e 正则本轮再收紧 —— 前置位统一 `(?:\b|[0-9$])`(覆盖
    `1w /etc/x` 地址连写),w/r/R **去掉路径锚**(`1w ../.zshrc`
    相对路径 —— run_shell 默认 cwd=~/.kaya,`..` 直达 $HOME,与
    trusted download 链成写 ~/.zshrc 的无确认 RCE;`w/etc/x` 无空格
    连写 —— 一律拒,不看目标形态),e 放宽为 `\s*\S`。safety 层补
    同源 DANGEROUS 兜底(`DANGEROUS_SED_SCRIPT_WRITE_READ_RE`,awk
    有同款先例)。刻意接受的 fail-closed 误杀(两侧钉死测试):
    `sed 's/a/e x/'` / `sed 's/wow/x/'` / `sed 's/error/ERROR/g'` 这类
    替换文本以 w/e 紧贴 `/` 开头的形态;正常 `s/foo/bar/` /
    `sed -n '1,5p'` 不受影响。
  - **trusted download URL host 校验(H-R2-2)**:① host 数字形态
    归一缺口 —— curl 按 inet_aton 语义解析,`http://127.1:8080/`
    (短点分)/ `http://2130706433/`(十进制)/ `http://0x7f000001/`
    (十六进制)都是 127.0.0.1,但此前只认标准点分四段 → 非标准
    数字形态一律拒(fail-closed);② 私网段一律拒:RFC1918
    (10/8、172.16/12、192.168/16)+ 169.254/16 + CGNAT(100.64/10)
    + unspecified + broadcast + 文档保留段 + v6 的 fc00::/7(ULA)/
    fe80::/10(link-local)/ v4-mapped(`::ffff:127.0.0.1`)—— 内网
    服务捕捞同样是 SSRF,取舍是 trusted download 从此不能打内网
    地址;③ `--resolve` / `--connect-to` 移出 trusted download 旗标
    白名单(URL host 与实际连接目标脱钩可绕过 URL 校验;非 trusted
    的 curl 本就整体落 network-read 黑名单,不受影响)。
  - **METADATA 基命令的子命令执行面(M-R2-1 + R3/R5 补漏)**:`rustup run` /
    `npm exec` / `npm dlx` / `npm run` / `npm run-script` /
    `npm explore` / `npm init|create`(等价 `npx create-*` —— 拉包并
    立即执行 bin;pnpm 同)/ `cargo run|build|test|bench|install|script` /
    `pip download` / `brew ruby` / `brew irb` 一律拒;R5 起 yarn 非纯
    查询一律拒(白名单反转 —— yarn classic 裸 script 简写 `yarn build`
    同样是执行面,裸 `yarn` 即 install),`bun` 整体移出白名单
    (JS 运行时,`bun -e` / `bun --print` / `bun <file>` 任意执行)
    (`[policy:blacklist]` —— 分别是 toolchain 执行通道 / 包下载执行
    / package.json script / build.rs RCE / sdist 构建执行 / Homebrew
    内嵌 Ruby)。判定是「base 之后任意 token 精确命中 deny 词表」
    (fail-closed:全局值旗标前置如 `cargo --config x=1 build` 不得
    骗过);纯查询形态(`rustup show` / `npm ls` / `npm view` /
    `cargo --version` / `pip show` / `brew info`)不受影响。刻意误杀:
    `brew info ruby`(同名 formula 查询)。Windows 备注:npm 是
    `npm.cmd`,`table_base_key` 只剥 `.exe` 不剥 `.cmd` → `npm.cmd
    exec` 落 allowlist-miss 拒(fail-closed 已覆盖)。
  - **grep 递归检测变形(M-R2-2)**:grep 族 value-flag 表补 `-d` /
    `--directories`(值 `recurse` ≡ `-r`,空格 / 内联 / glued 三形态
    都认,`-d skip` 等非 recurse 值不误判);长旗标递归判定改为 GNU
    getopt_long 前缀缩写语义(`--r` … `--recursive` 与
    `--dereference-recursive` 的合法前缀,`--r` 歧义按递归
    fail-closed)。
  - **wc `--files0-from`(L-R2-3)**:值(FILE 是文件清单,wc 再逐个
    读清单里的文件)收进路径提取过 trusted-root 判定(内联与空格
    两形态),此前不在 value-flag 表、内联形态被跳过 → 路径集为空
    被当 stdin-only 放行。

**为什么 trusted read 不会泄漏**:scrubber 仍跑,被读出的内容里如果
有 AWS key / API key / PEM,继续 redact。审计 trail 的
`ToolCallCompleted` 事件标 `stage: "trusted_read"`,judge 规则跳过
"reader-without-redaction"。

**escape 拦截**:`..` 逃逸(归一化前/后比较)、前缀相似
(`kaya-other/` 看似 `kaya/` 的子目录,实际不是)
在服务端 tokenize + 归一化 + 前缀检查拦下。

### 5.1.1 Policy 三表 + 7 个 deny 类别(Round-67, 2026-07-05)

把原来的"黑/白名单"二元结构改造成 **3 个语义表 + 永久 blacklist +
allowlist-miss 兜底**,LLM 收到的 deny reason 现在带**结构化前缀
`[stage:subcategory]`**,能精准对应到下面 7 个类别之一。系统 prompt
(`policy_prompt::render_allowed_section_*`)与 run_shell 工具描述
(`tools/run_shell.rs::rules_prose`)从同一份数据源生成,**单一真相,
无法漂移**。

| 表 | 命令 | 何时放 | 何时拦 |
|---|---|---|---|
| **METADATA** | `ls` / `ps` / `netstat` / `df` / `wc` / `find` / `uname` / `whoami` / `date` 等 | 永远 | 永不(v2.1.10 review 起例外:`wc` / `md5` / `shasum` / `sha256sum` 带 root 外文件路径参数时按 trusted-read-miss 拒——内容/大小 oracle;Windows flavor 下参数含 `env:` / `$env:` 引用即拒——环境变量直出;R2/R3/R5 起:`command` 只放行 `-v/-V <name>` 纯查询,`rustup run` / `npm exec|dlx|run|run-script|explore|init|create` / `yarn` 非纯查询一律拒(白名单反转,裸 script 简写也拒)/ `cargo run|build|test|bench|install|script` / `pip download` / `brew ruby|irb` 子命令执行面按 [policy:blacklist] 拒;`bun` 整体移出白名单(R5:JS 运行时,`bun -e`/`--print`/`<file>` 任意执行,与 node/python/deno 对齐落 allowlist-miss)) |
| **STREAM_TRANSFORMERS** | `head` / `tail` / `sort` / `grep` / `awk` / `sed` / `cut` / `tr` / `uniq` 等 | 非首位 + 无显式路径参数永远(2026-08-20 起:管道位带显式路径参数按 trusted-read 门控;`tr` 例外 —— 不接受文件操作数,永远 stdin-only);首位无文件参数放;首位 + 参数文件在 `~/.kaya/` 放 | 首位 + 文件不在 `~/.kaya/`;管道位 + 显式路径不在 `~/.kaya/`;`sed -i`(写)永远拒;sed script 内 `w` / `r` / `R` / `e` 命令永远拒(v2.1.10 review H1);无路径的 `grep -r` / 裸 `rg` 等递归搜索在 cwd 出 root 时拒(v2.1.10 review M-sec-2) |
| **FILE_READERS** | `cat` / `bat` / `view` / `less` / `more` / `xxd` / `hexdump` / `od` / `strings` / `base64` / `base32`(`source` 于 v2.1.10 review C1 移出 —— 执行不是读,一律 allowlist-miss 拒) | 参数文件在 `~/.kaya/` | 不在 |
| **永久 blacklist** | shell escape / `eval` / `exec` / `xargs` / `curl` / `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` 等(`rm`/`sudo` 属 safety DANGEROUS、`osascript` 属 safety OPERATION、`tee` 落入 allowlist-miss,均不在此表) | 永不 | 永远 |

**7 个 deny 子类别**(对应 `[stage:subcategory]` 前缀):

- `[safety:dangerous]` 破坏性操作(`rm -rf` / `sudo` / `dd if=` / `mkfs` / `kill -9`)
- `[safety:repl]` REPL/交互式(`vim` / `less` / 裸 `python` / 裸 `node` / `irb` / `gh repl`)
- `[safety:operation]` 系统变更(`npm install` / `brew install` / `systemctl` / `launchctl`)
- `[policy:blacklist]` 永久黑名单(shell escape / eval / exec / xargs / curl / wget / env dump / pbcopy / screencapture / open / history)
- `[policy:allowlist-miss]` 命令不在任何 3 个白名单表
- `[policy:trusted-read-miss]` 文件读取器/流转换器路径不在 `~/.kaya/`
- `[policy:trusted-download-miss]` `curl -o` 目标不在 `~/.kaya/` 或扩展名不安全(`.sh` / `.py` / `.exe` / `.dmg` / 二进制 / 安装包)

**隐私面分析**:本变更**不扩大数据泄露面**。`ps aux | grep nginx` 这类
合法 stream filter 行为,旧模型下要靠"PIPELINE_CONSUMERS 跳过
blacklist"的隐式规则;新模型下它们从 blacklist 升级到
STREAM_TRANSFORMERS 的非首位位置,**显式**合法。同样,sed `-n`(只读)
也在表里放行,sed `-i`(写)明确拒。

**Rust ↔ TS 同步**:`src/lib/runShellPolicy.ts` 与 `src/lib/safety.ts`
是 audit judge 用作 deterministic replay 的镜像 — 任何 Rust 端增删
表项必须**同 PR** 同步 TS 端,否则 audit judge Pass 1 会把每一个
录制的 session 都报 `decision-mismatch` / `stage-mismatch`。详见
[`docs/superpowers/specs/2026-07-05-policy-llm-alignment-design.md`](./superpowers/specs/2026-07-05-policy-llm-alignment-design.md)。

### 5.2 Safety(危险命令检测)

源码: `crates/kaya-core/src/llm/safety.rs`(DANGEROUS / REPL / OPERATION 三套 regex;测试数见 §11.1)

- **DANGEROUS**:`rm -rf` / 磁盘 / 设备直写 / `dd if=` / `mkfs` 等
- **REPL**:裸调 `vim` / `less` / `python` / `node` / `irb`(会进交互)
- **OPERATION**:`sudo` / `brew install` / `kill` / `launchctl` / `diskutil
  eraseDisk` 等
- **递归深度 3**:heredoc / `$(...)` / `` `...` `` / `eval` / `bash -c`
  的内层 body 也扫,防"LLM 合成一段脚本塞进 heredoc 让你执行"
- **默认拒绝**:不匹配白名单 = 拒

**用户在 terminal 里直接输入的命令不经过这两道门** — 用户对自己的
shell 行为负责。这是设计决策,不是漏洞:用户输 `rm -rf` 不会被拦,
输 `kill` 不会被拦,我们**不**替用户做这个决定。

### 5.3 Windows 语义层(Windows 移植 Phase 1b, 2026-08-24)

Windows 上 `run_shell` 子进程是 PowerShell(Phase 1d),而 Windows
PowerShell / pwsh 自带大量 Unix 风格 alias(`cat` → Get-Content、
`ls` → Get-ChildItem、`rm` → Remove-Item 等),**Unix 命令名在
Windows 上依然可执行**;反过来 Windows 危险 cmdlet 在 Unix 不存在,
匹配到也无害。据此:

- **三表**:「Unix 表原样 + Windows 追加并集」。`OsFlavor` 新增
  `Windows` variant;`allowed_bases_for(flavor)` /
  `stream_transformers_for(flavor)` / `file_readers_for(flavor)` 按
  flavor 构造(Macos → CORE ∪ MACOS_EXTRA;Windows → (CORE −
  `type` − `find`) ∪ WINDOWS_EXTRA;Linux → CORE)。**`type` 例外**:
  Unix 是纯 metadata builtin,pwsh 里是 `Get-Content` alias(读
  文件),Windows 上从 METADATA 剔除改收 FILE_READERS。**`find`
  例外**(2026-08-24 review):Unix 是结构查询,Windows 上 find.exe
  是 **grep 语义(读文件内容)**,Windows 上从 METADATA 剔除落
  allowlist-miss(pwsh 正解 `Select-String` 已在流变换器表);
  macOS / Linux 的 `find` 保留不动。Windows 追加:
  METADATA 收 `where` / `tasklist` / `ipconfig` / `systeminfo` /
  `Get-Process` / `Get-ChildItem` / `Get-Location` 等只读 cmdlet;
  STREAM_TRANSFORMERS 收 `Select-Object` / `Sort-Object` /
  `Select-String` / `Where-Object` / `Compare-Object`(**不收**
  `ForEach-Object` — scriptblock 代码执行;**不收** `Tee-Object` —
  与 tee 同款 T-junction 写盘);FILE_READERS 收 `Get-Content` /
  `type` / `Format-Hex`。PowerShell 命令名大小写不敏感:Windows 表
  条目小写存储,查表前 base 归一小写(`table_base_key`)。
  2026-08-24 收尾:METADATA 追加 `Get-Command`(= which 的 pwsh
  语义正解 —— `where` 在 pwsh 里是 `Where-Object` alias,运行时会
  错;`where.exe` 显式形态仍在表内);`table_base_key` 的 Windows
  归一追加**剥结尾 `.exe`**(大小写不敏感,`where.exe` /
  `Get-Content.exe` 与裸名同义命中;**Unix 不剥**,`.exe` 在 Unix
  是合法文件名字符)。剥后缀的 key 只用于查 allowlist/三表,不改变
  实际执行的命令,allowlist 条目全是只读语义,无提权面(评估详见
  policy.rs `table_base_key` 注释)。
- **黑名单 / safety 危险面**:两平台收同一份**全局并集**(不按
  flavor 分)。policy 永久黑名单追加 `iex` / `Invoke-Expression`、
  嵌套 `powershell` / `pwsh`(整条命令名,覆盖 `-enc` /
  `-EncodedCommand` base64 隐匿执行的一切形态)、`cmd /c|/k`、`wsl`、
  `Get|Set-Clipboard`(对齐 `pbpaste`/`pbcopy`)。safety DANGEROUS
  追加 `format <盘符>:`(锚定盘符,避免误杀 `git format-patch`)、
  `diskpart` / `bcdedit` / `reg add|delete` / `Set-ExecutionPolicy` /
  `Stop|Restart-Computer` / `Clear-Disk` / `Remove-Item -Recurse
  -Force`(对齐 `rm -rf`);OPERATION 追加 `Stop|Restart-Process`、
  `Stop|Start|Restart-Service`、`Set|Add-Content` / `Out-File`、
  `New|Rename|Move|Copy-Item`。
- **trusted root 路径归一化**(安全不变量,fail-closed):
  `%USERPROFILE%` 与 `$env:USERPROFILE` 展开(大小写不敏感,其余
  形态不展开即拒);`\` / `/` 统一归一为 `/`;盘符与整个路径比较
  大小写不敏感(NTFS 语义,Unix 保持敏感);UNC(`\\server\share`)
  一律 deny;`..` 逃逸沿用归一化后前缀检查;`.kaya-evil` 类前缀
  误判沿用「精确相等 或 root + `/`」边界检查;**8.3 短名**
  (`KAYA~1`)无法纯字符串判定 —— 以「归一后任何路径段含 `~`
  即拒」收紧兜底(Windows-only 规则,macOS 行为不变)。空设备
  例外按 flavor:Unix `/dev/null`(输入侧加 `/dev/zero` /
  `/dev/urandom`),Windows `NUL` / `$null` —— Windows 形态不并
  进 Unix(`NUL` 在 Unix 是普通文件名,放行会让 `echo x > NUL`
  在 cwd 写盘)。归一化全部做成 flavor 参数纯函数,Windows 用例
  在 macOS host 上全量可测。
- **heredoc / `$(...)` 扫描评估结论**:PowerShell 无 heredoc 语法,
  既有扫描对 PowerShell 命令串**无误报**;`$(` 在 PowerShell 是子
  表达式(执行通道),Layer 0a 继续硬拒;**反引号在 PowerShell 是
  转义符**不是命令替换 —— Windows flavor 不再把反引号当命令替换
  拦截(否则 `Write-Output "a`nb"` 误杀)。漏报面(PowerShell
  here-string `@"..."@`)不强行补,由 allowlist-miss catch-all 兜底。
- **PowerShell 执行形态一律硬拒**(2026-08-24 review,fail-closed):
  Windows flavor 下 Layer 0a 把未加引号的 `(` 与 `$(` 同级硬拒 —
  — `(...)` 分组子表达式与 `@(...)` 数组子表达式都是**先求值再当
  参数**的合法 PowerShell 语法(`Write-Output (Get-Content file)`
  此前绕过整个 gate)。双引号内裸 `(` 是字面量不拦(PowerShell
  双引号只插值 `$(` 与反引号);误杀 `(Get-Date).Year` 类合法形态
  是刻意取舍,内部命令拆成独立调用即可。同批:**`\` 在 PowerShell
  是路径分隔符不是转义符**,命令替换扫描器与重定向扫描器
  (strip_unquoted_redirects)的两字符反斜杠跳过在 Windows flavor
  全部禁用 —— 否则 `C:\$(Get-Clipboard)` 的 `$(`、`Get-Date \>
  out.txt` 的 `\>` 被当转义对吃掉,分别造成任意读 / 任意写
  fail-open。Round 2 起分段器(split_segments)同样禁用跳过:
  `;` / `|` 在 PowerShell 不要求前后空白,`Write-Output x\;Get-
  Content C:\secret` 的 `\;` / `\|` 转义形态曾把整条命令吞成单段
  (base ∈ METADATA 放行,被吞的 reader 永远扫不到),现同样硬拒。
  Unix 行为逐字节不变(zsh 的 `\` 转义语义与裸 `(...)`
  语法错误性质不变)。
- **gate 显式 flavor 化**(2026-08-24 review):`check_run_shell_policy`
  拆出 `check_run_shell_policy_for(command, ctx, flavor)` 薄壳,
  gate 全程(三表 / 查表 key / basename / 命令替换与重定向扫描 /
  trusted read+download / trusted-root 归一)按 flavor 透传,不再
  内部硬编码 `OsFlavor::current()` —— Windows 分支的端到端回归
  测试在 macOS host 上直接可跑,与 TS 镜像矩阵对齐。
- **cmdlet 旗标形态**:路径提取识别 PowerShell 的三种旗标赋值形态
  —— `-Path value`(空格)、`-Path:value`(冒号内联)、`-Path=value`
  (等号内联,2026-08-24 补;此前 `Select-String -Path=C:\secret`
  的 `=` 形态不识别导致路径集为空、被误判 stdin-only 放行,是
  fail-open,已修复并加回归矩阵)。`Get-Content` / `type` /
  `Format-Hex` 的 `-Path=` / `-LiteralPath=` 值同样当路径过
  trusted_root 校验;`-Tail=5` 类「值不是路径」的旗标不误判。
  **旗标前缀缩写**(`-pa` = `-Path`)不识别 —— 按未知旗标跳过,
  由无路径判定(Layer 6 拒)/ allowlist-miss 兜底,属已知残余风险。
- **scrub safe-name**:env-var scrub 的关键词正则(`KEY` / `SECRET`
  / `TOKEN` / `PASSWORD` / …)自 Round-58 起没有 safe-name 白名单
  (旧 deny-list 是死代码已删);Windows 系统变量(`USERPROFILE` /
  `APPDATA` / `PSMODULEPATH` 等)均不含上述关键词,天然不会被
  redact,**无需新增任何条目**。

---

## 6. 第三关:Execute 隔离

源码: `crates/kaya-core/src/llm/tools/run_shell.rs`(stage 3)

AI 调 `run_shell` 时,Rust **独立** spawn 一个 `zsh -f -c <cmd>` 子进程:

- `-f` 抑制 `~/.zshrc` 加载 — 这是**关键**。如果不抑制,用户在 rc 文件
  里定义的 alias `ls='rm -rf $HOME/.cache'` 会被展开,policy 检查的是
  字面命令名(`ls`),但实际执行的是 `rm -rf`。
- **不走用户的 PTY**。完全独立的子进程,失败不影响 xterm 状态。
- **默认 8 秒超时**(`DEFAULT_TIMEOUT_SECS`;用户可经 `run_shell.timeout_secs`
  配置;`du` / `find` / `grep` 等慢扫描命令下限 30s,`SLOW_SCAN_MIN_SECS`)
  + **100KB 输出上限**。超时直接 kill。
- **无网络状态保留**:不像用户在 shell 里累计的 venv / conda / alias,
  AI 的子进程每次都是干净的 zsh — 副作用不污染用户环境。

> Windows 移植 Phase 1d(2026-08-24):Windows 上子进程是
> `pwsh/powershell -NoProfile -Command <cmd>`(参数形态由
> `pty::env::shell_exec_args` 按 shell 文件名分派)——`-NoProfile`
> 不加载 `$PROFILE`,与 `-f` 同设计意图;超时杀进程走
> `taskkill /PID <pid> /T /F` 进程树(unix 是 killpg 进程组);
> boot 环境捕获(`capture_user_env`)整体跳过,直接用进程 env
> (Windows GUI 启动无 launchd env 缺失问题)。五道关卡结构不变。
> W1.3(2026-08-25)起 Windows 下 `-Command` 前注入
> `[Console]::OutputEncoding=UTF8` prelude(gate 之后注入,不影响
> policy/safety 语义),管道字节解码加 GBK 兜底(中文系统 native exe
> 本地化输出)。

---

## 7. 第四关:Output Scrub(凭据脱敏)

源码: `crates/kaya-core/src/llm/scrub.rs` + TS mirror
`src/lib/scrubber.ts`

### 7.1 Round-54 (2026-06-25) 三层 Tier 设计

把 ~38 类 secret / PII 模式按"是不是必须不能漏"分成三层(v2.1.10
review R2 对照 gitleaks 常见规则补了 8 类,见 Tier 0 列表末尾):

**Tier 0 — Core(所有档都跑,Minimal / Balanced / Strict 都启用)**

核心 secret,绝不能被档位门控 —— 用户选"strict"(UI 最弱)也
不能漏掉 API key / PEM 私钥。~28 类:

- Provider tokens: AWS(`AKIA[16 chars]` + `AWS_ACCESS_KEY_ID=` /
  `AWS_SECRET_ACCESS_KEY=` 前缀两种形式) / GitHub PAT(4 个子类:
  `ghp_` / `gho_` / `github_pat_` / `(ghu|ghs|ghr)_`) / GitLab PAT
  / Anthropic / OpenAI (`sk-proj-`) / DeepSeek / Moonshot
  (`sk-[32+ chars]`) / Stripe (`sk_live_` / `sk_test_`) / Slack /
  Telegram bot / Tavily (`tvly-[A-Za-z0-9_-]{20,}`) /
  **v2.1.10 review R2 补齐(gitleaks 对齐)**:Slack app-level
  (`xapp-<数字>-…`) / npm (`npm_`+恰好 36 位) / PyPI
  (`pypi-AgEIcHlwaS5vcmc` 固定前缀+50+ 字符) / SendGrid
  (`SG.`+恰好 22+43 两段) / DigitalOcean (`dop_v1_`+恰好 64 位
  小写 hex) / Shopify (`shpat_`+恰好 32 位 hex) / Twilio
  (`SK`+恰好 32 位小写 hex) / 三段式 JWT
  (`eyJ…​.…​.…`,每段 ≥8 字符,类别 `jwt`)
- 通用 `api_key=xxx` 形式 (有 fake-key 过滤: `xxx` / `changeme` /
  `placeholder` 等不误吃)。**2026-08-20 review C3**:值前加可选引号
  `['"]?` —— `~/.kaya/agent-providers.json` 存 `"api_key": "sk-…"`
  (JSON 形式),旧的 `:` 形式正则被 `:` 与值之间的 `"` 阻断,整条 JSON
  值会经 trusted-read `cat` 漏给 LLM。Rust `API_KEY_RE` 与 TS
  `scrubber.ts` 对齐(TS 原本就有 `['"]?`),parity 测试
  `scrubs_api_key_json_quoted_value` 锁两侧。
- Bearer token: 合并了 `Authorization: Bearer eyJ...` 和裸
  `eyJ...eyJ...` 两种形式,统一类别名 `bearer-token`。**v2.1.10
  review R2**:Rust 侧补齐裸 `eyJ…eyJ…` 备选分支(此前 Rust 无此
  形态,裸 JWT 只靠 entropy 兜底拦头段,`.` 切开的 payload /
  signature 段会漏);payload 非 `eyJ` 开头的三段式 JWT 由新增的
  `jwt` 类别兜底(两侧一致)
- URL credentials (http(s)://user:pass@)
- PEM 私钥整块 (BEGIN + body + END,7 种 type 变体)
- DB URL password (`mysql://` / `postgres://` / `mongodb://` /
  `redis://` / `amqp://` / `rabbitmq://` / `sftp://` / `smtp://` 携带
  user:pass@)
- High-entropy fallback (`\b[A-Za-z0-9+=]{20,}\b` + 类多样性 +
  至少 2 个不同字符 + 不是 PEM block + 不是自然语言)

**Tier 1 — Env(balanced + aggressive 跑)**

- `KEY=value` / `export FOO=bar` 模式 (有 `PATH` / `HOME` 等安全
  key 的 deny list,只 redact 真敏感的)
- 云厂商 key: GCP (`AIza[35 chars]`) / Azure (`AccountKey=` + 86-88
  base64) / Alibaba (`LTAI[16+ chars]`)

**Tier 2 — PII(aggressive 档独有)**

- email
- phone (CN `1[3-9]\d{9}` + intl `\+[1-9]\d{6,14}`)
- US SSN (`\d{3}-\d{2}-\d{4}`)
- credit card (过 Luhn 校验才视为真卡,16 位硬件 UUID 不会被误吃)
- id-cn 中国身份证号(17–19 位数字串 + 18 位 X/x 校验位形,整 span 脱敏;round-58 去掉校验和门、round-72 放宽到 17–19 位、2026-07-20 review 修复 Rust 侧 19 位泄漏 + X 校验位同 span)。**注册表顺序:id-cn 在 credit-card 之前**——同串同时过 GB-11643 校验位与 Luhn 时类别判 id-cn,Rust(`tier2_pii`)与 TS(`scrubber.ts` registry)顺序必须一致(两侧各有交集 fixture 锁 `1101052000010100008`;drift → audit-judge category-mismatch)

### 7.2 三档严格度 — 命名是反的(故意的)

| 前端标签 | Rust 端 | 触发范围 |
|---|---|---|
| `aggressive` (默认) | `Strict` | Tier 0 + 1 + 2 |
| `balanced` | `Balanced` | Tier 0 + 1 |
| `strict` | `Minimal` | Tier 0(不再仅跑 high-entropy) |

**high-entropy 永远最后跑(所有档)**:2026-07-20 review 修复前 Rust 把
`scrub_high_entropy` 放在 tier0 **开头**附近(先于具体 pattern),与 TS
镜像(registry 最后一项)优先级相反 → 含 ID 的 20+ 字符长串两侧类别分裂
(Rust `entropy-string` / TS `id-cn`,audit-judge category drift)。现在
Rust 在所有档的 tier 派发**之后**跑 entropy——具体 pattern (id-cn /
provider key 等) 先赢类别,entropy 兜底,两侧逐形状对齐(回归锁:
`crates/kaya-core/tests/id_cn_entropy_mirror.rs` ↔
`scrubber.test.ts` "round-2 mirror parity")。

**Round-54 行为变更(隐私优先)**:

- **默认值改为 `aggressive`**:pre-round-54 的 `Balanced` 默认会让
  PII 漏,违背 privacy-first 定位。CLI / 桌面端 / 未知字符串兜底
  全部跟 `aggressive`。
- **核心 secret 模式在所有档都跑**:pre-round-54 时,`aws-access-key` /
  `private-key` / `bearer-token` / `github-pat-*` / `slack-token`
  在 Rust 端的 `Balanced` 档才启用,`Minimal`(用户"strict")档会
  漏这些 —— TS 端则在 `strict` 档就跑,两端行为不一致。round-54
  把所有核心 secret 提到 Tier 0,统一两边行为。
- **store v18 硬迁移**:桌面端 `localStorage.scrubLevel == "balanced"`
  / `"strict"` 的存量用户被自动升级到 `"aggressive"`(`store.ts`
  v17→v18 迁移 body)。CLI config.toml 显式写过的 `scrub_level` 不
  被覆盖 —— CLI 尊重显式配置,只改默认值。

`scrub.rs::frontend_name_to_scrub_level` 是单一真相源。命名是
**反**的,因为 Rust 端以"严格 = 拦更多"为基准,前端 UI 标签
"aggressive = 拦更多"是用户视角。两边命名合理,反着翻译才一致。
**修改这一对映射时,README / CLAUDE.md / scrub.rs 头注释三处都要改**。

### 7.3 Round-55 (2026-06-26) 跟进 — TS/Rust regex 形态对齐补丁

Round-54 落地后,code review 发现 9 处 TS / Rust pattern 形态不齐。
这一节列出对 §7.1 Tier 设计有强化的补丁。完整 spec + 测试见
[`docs/superpowers/specs/2026-06-25-scrub-default-aggressive-design.md` §11](./superpowers/specs/2026-06-25-scrub-default-aggressive-design.md)。

**对 §7.1 Tier 0 的强化**(都是 Tier 0 直接关系到安全的):

- **`env-var-secret` (Tier 1) 多行匹配**:之前 regex 没有 `m` flag,`^`
  只匹配 input 起头,`printenv` / `env` dump 第 2 行起的 secret
  全部绕过。round-55 加 `m` flag(TS) / `(?m)` (Rust),每一行
  都过扫描 + deny list。`API_KEY=sk_live_xxx\nSECRET_KEY=yyy`
  这种 dump 现在正确 redact。
- **`aws-access-key` 加 `AWS_SECRET_ACCESS_KEY=` 前缀支持**:之前 TS
  regex 只匹配裸 `AKIA[16 chars]` 或 `AWS_ACCESS_KEY_ID=AKIA...`,
  真正的 40-char `AWS_SECRET_ACCESS_KEY=<base64>`(没 AKIA 前缀)
  漏。Rust 端从 round-7 起就有。round-55 TS 端加上。
- **`deepseek-key` 加 `\b` 边界 + `_`/`-` 字符**:之前 TS regex
  `/sk-[A-Za-z0-9]{32,}/` 没有 word boundary,内部带 `-` 的 token
  会漏。Rust 端从 round-7 起就有。round-55 TS 端加上。
- **`password-in-url` 支持空用户名**:`redis://:pass@host`(空
  用户名,Redis 无 ACL 模式)之前漏,现在匹配。
- **11 个 Tier 0 / Tier 1 pattern 加 `\b` 边界**(`githubPat*` /
  `gitlabPat` / `anthropicKey` / `stripeLive/Test` / `slackToken` /
  `gcpApiKey` / `alibabaAccessKey`):被标点(括号 / markdown fence /
  HTML tag)包的 token 之前被错误归类,round-55 加 `\b` 后只匹配
  真正的 token 边界,audit-judge 的 deterministic replay 不再
  报 `category-drift`。
- **`openai-key` 窄化到 `sk-proj-` 前缀**:之前 `/sk-(proj-)?.../`
  把 bare `sk-` 32+ 字符(其实是 deepseek 形状)归到 openai-key 桶,
  audit-judge replay 报 `openai-key` vs Rust `deepseek-key` 漂移。
  round-55 强制 `sk-proj-` 前缀,与 Rust `OPENAI_PROJ_KEY_RE` 对齐。
  **真正的 OpenAI legacy `sk-` key 会落到 deepseek-key 桶**(secret
  仍被 redact,只是分类标签不同 —— 这层 trade-off 在
  [`docs/superpowers/specs/2026-06-12-llm-secrets-redaction-design.md`](./superpowers/specs/2026-06-12-llm-secrets-redaction-design.md)
  Pattern 库表的 `deepseek-key` 一行说明里)。
- **`azure-storage-key` 加 `/i` flag**:小写 `accountkey=`(常见于
  `kubectl get secrets -o yaml` 输出)之前漏,现在匹配。
- **`slack-token` 字符类去掉 `o`**:之前 `[aboprs]` 含 `o`,会误匹
  配不存在的 `xoxo-...` token。Rust 端从一开始就是 `[baprs]`。

**测试强化**(11 个新增 tripwire):

- 9 个 `\b` boundary 回归测试:每个 Tier 0 pattern 各 1 个,锁定
  "被标点包时不归类到具体桶,只走 high-entropy 兜底"
- 1 个 `azure-storage-key lowercase` 回归测试
- 1 个 `slack-token xoxo-` 否定回归测试

**测试覆盖现状**(2026-08-04 重测):

- TS:2457 tests pass(71 spec files)
- Rust:测试按模块持续扩张;隐私关键模块的精确数量见 §11.1

(历史 round 的完整仓库总数容易受新增非隐私测试与已知并行 PTY flake 影响;本节只把 `npm test` 的稳定全量结果和 §11.1 可静态核对的模块数量作为当前基线。)

### 7.4 关键 invariant:整段 PEM 整块 redact

见 §4.3。Rust 用 `once_cell::Lazy<Regex>` 编译一次,7 种类型变体的
BEGIN/END 对作为 alternation。CLI / desktop / audit judge 三处共用
同一份 `ssh_key_regex()` 访问入口(2026-06-23 round-7 dedup),加新类型
时只改一处。

**Round-54 新增**:high-entropy fallback 会跳过 `-----BEGIN ... KEY-----`
整块 (public + private)。公开公钥 body 是 base64 形态(看起来像
高熵),但本就是要公开分享的;私钥 body 已经被 `private-key` 整块
redact,不需要 high-entropy 再吃一次。`PEM_BLOCK_RE`(在 Rust
和 TS 镜像都有)在 `looks_like_token` 之前先 filter 掉所有 PEM span。

### 7.5 Placeholder 格式统一

Rust + TS 两端都输出 `[REDACTED:<category>]`(2026-06-19 后统一),LLM
知道"丢了什么类型",仍能继续推理。Pre-2026-06-19 Rust 端是
`[<category>]`,前端是 `[REDACTED:<category>]`,会导致 conversation
JSON / debug JSONL / UI 显示格式不一致 — 改成统一格式修了这条。

### 7.6 外部 API 工具（round-58, 2026-06-29）

`kaya` 的 LLM 可通过 Tavily 联网搜索和抓取网页（`tavily_search` /
`tavily_extract` 工具）。所有 Tavily 返回的外部内容**默认走用户当前 `scrubLevel`
过滤**（默认 `aggressive` = Tier 0+1+2 全开），避免外部网页中潜在的 API key /
PII / 密码泄露给 LLM。

**不可信假设**：Tavily 返回的内容视为外部不可信输入，即使经过 scrub 也不能保证
100% 干净（regex 抓不到 base64 / 加密 blob）。建议 LLM 引用时人工核对关键事实。

### 7.7 terminal_read(终端输出只读可见,2026-08-09)

chat 的 `terminal_read` 工具可以列出你的终端 tab 并读取指定 tab 最近的输出
(每 tab 64 KiB 去 ANSI ring buffer,仅内存,不落盘;全屏 TUI(vim、AI CLI 等)
运行期间,捕获切换为实时屏幕快照——由内置终端模拟器(vt100)在内存中重建画面,
读取的是当前屏幕文本与光标位置,同样仅内存、不落盘,送达 AI 前与所有工具
走同一 scrub 管道)。

- **何时脱敏**:终端文本只在 LLM 调用该工具时、送达 LLM 之前经 `scrub_text`
  过滤——与 run_shell / list_dir 同一管道、同一隐私档位。你在 Settings 调整
  secret redaction level 对它同样生效。
- **cwd 不脱敏**:tab 列表展示每个 tab 的 cwd,属进程元数据(与 run_shell 的
  cwd/trusted_root 同一类别,本就不脱敏),但项目目录**名**本身可能嵌入敏感
  字符串——把秘密放进目录名的用户应当知悉。
- **固有局限**:scrub 基于模式匹配,挡不住所有秘密。在终端进行高敏操作
  (查看私钥、生产凭证等)期间,避免让 AI 读取终端,或临时把档位调到 aggressive。
- **只读边界(默认)**:默认情况下(未授权时)AI 不能向你的终端写入任何内容;
  terminal_read 没有写通道;授权后的写入见下节。

**授权写入(2026-08-16,P1;2026-08-17 加总闸)**:前提是你已打开顶栏的 **coding 模式开关**——它是「驾驶」总闸门,关闭时 `terminal_write` 工具根本不注册,chat 连向终端 AI 发输入的能力都不存在;打开之后,再经你在对话中的授权卡片逐 tab 允许,chat 的 `terminal_write` 工具才可向运行白名单 AI CLI 的 tab 发送输入。授权存内存、绑定 (tab, agent 进程),进程退出或 tab 关闭即失效,app 重启清空。每次授权请求与写入记入本地审计日志(元数据:tab、agent、pid、字节数;外加经 secret 脱敏后、截取前 200 字符的输入预览,key 写入记 null)。识别前台进程在 macOS 上靠 `tcgetpgrp` + `ps` 快照,可精确判定「当前前台进程组」是否为白名单 CLI;Windows(2026-08-26 起)改为 sysinfo 进程快照 + 从该 tab shell 子进程向下做进程树子树匹配(`.exe` 后缀归一,ConPTY 无前台进程组概念,以「会话进程树内存在白名单 CLI」近似——这会把同一 shell 子树内的后台 agent 也判定为可写,是比 macOS 更宽松的安全边界),数据不出本机。R3 加固(2026-08-31):① text 参数含 ESC 等终端控制字符(除 Tab/换行外的 C0/C1)一律 `[safety:control-chars]` 拒——防授权后借 agent 回显/TUI 解析链注入终端控制序列(改窗口标题 / OSC 52 剪贴板等),控制序列只能走受枚举约束的 key 参数;② 结尾换行的提交键 `\r` 与正文隔 500ms 分两次写(paste-burst 兜底会吞连发回车),`\r` 发出前再做一次同标准的前台进程 + 授权复验——500ms 窗口内 agent 退出则 fail closed,回车不会落进普通 shell。

**数字应答写入(interaction_answer,2026-08-16/17)**:terminal_write 之外的第二条「向终端写入」通道。coding 模式下,TUI 审批菜单的数字选项随事件下发;你在 chat 输入框敲菜单序号,即把所选键写回 TUI。仅收个位数字 1-9;Rust 侧四道复验(tab 存活锁内重读 / 前台进程白名单 driving_roster / 按最新屏幕快照重取菜单 / 选项合法)全部通过才写入。它不是 terminal_write 的旁路,不消耗 LLM 驾驶授权——敲数字是你本人的显式动作;LLM 永不代为回答。

**未知 /命令 转发(agent_command,2026-08-18)**:第三条终端写入路径。chat 输入未知 `/命令` 且当前会话已绑定 agent 时,原文转发到该 agent 的终端输入行并回车;绑定是会话级内存态(`store.agentTarget`),不写盘。后端 `run_agent_command` 镜像 interaction_answer 的 fail-closed 复验,`validate_agent_command_text` 拒绝控制字符(含 Unicode 双向/零宽字符 U+202A-202E / U+2066-2069 / U+200B-200F / U+FEFF —— Trojan Source 视觉欺骗面,R3 加固)与超限文本;转发成功记一条 user-input 审计(过 input guard)。

**等待感知(2026-08-16,B)**:tab 交互确认提醒的 watcher 为**只读**——只对画面快照做模式匹配,绝不向终端注入输入;通知卡片上的确认内容摘要(excerpt)经 secret 脱敏后仅用于本地展示与通知,不会发送给模型。**例外(coding 模式,2026-08-17)**:当你显式开启 coding 模式后,上述经脱敏的 excerpt 会作为 host 指令文本的一部分进入一次 chat 回合——即随该回合发送给你当前选用的模型,用于让 kaya 主动转达「哪个 tab 在等你决策 / 哪个任务已跑完」并向你询问指示。coding 模式默认关闭;关闭它即恢复「excerpt 不发送给模型」的默认行为。

**Agent 配置文件读写(2026-08-17,P2-F1)**:chat 的 `agent_config` 工具可读取/修改三个受支持编程 agent 的 CLI 配置文件:claude-code → `~/.claude/settings.json`、kimi-code → `~/.kimi-code/config.toml`、opencode → `~/.config/opencode/opencode.json`。
- **读取**:`action="read"` 返回的内容先做键名脱敏(key/token/secret/password 命名的键只回前 4 字符 + `***`),再整体过一遍 secret scrubber 兜底(非敏感键名下的 secret 也不会发给模型)。
- **修改需显式授权**:`action="edit"` 是持久写入——每次修改都必须经你在对话中的授权卡片确认后才落盘;授权卡片只展示目标文件路径与待合并的顶层**键名**(patch 的值可能含密钥,从不上卡片、不进事件)。授权存内存、绑定 (会话, agent),新聊天会话会重新询问,app 重启清空。未授权时模型只收到「等待用户确认」的提示,不会写入任何字节。
- **写前备份**:每次写入先把旧版本备份为 `<file>.bak`;深合并保留 patch 未提及的既有字段。
- **`hooks` 键一律拒绝**:该键可在 agent 后续运行时装订任意 shell 命令,经本工具的合并请求会被直接拒绝(连授权卡片都不会出现),需你手动编辑配置文件。
- **审计**:每次授权请求与每次成功写入各记一条本地 App 日志(`~/.kaya/logs/`,含 agent、文件路径、合并的顶层键名——**绝不含值**)。
- **命令行版(kaya-cli)**:无授权卡片界面,`edit` 一律自动拒绝并提示你手动编辑;`read` 可用且走同一脱敏管道。

**Agent 安装/更新/卸载(agent_manage,2026-08-21;卸载 2026-08-27)**:chat 的 `agent_manage` 工具是 Agents 面板「安装 / 更新 / 卸载」按钮的**聊天等价物**——你说「帮我装 claude-code」时,模型经它发起与面板按钮完全相同的后台任务(同一单任务闸、同一事件流、同一 App 日志)。
- **只执行 catalog 固定命令**:工具参数只有 `action`(status/install/update/uninstall 枚举)和 `agent_id`(内置 catalog 白名单校验),**没有任何自由命令参数**;实际执行的命令(npm/brew/script)完全由编译期内置 catalog 决定,模型无法注入自己的 shell。
- **为什么不经 run_shell 五道关卡**:那五道关卡(policy gate / safety gate / execute 隔离等)防的是 LLM 生成的 shell 文本;本工具没有 LLM 生成的命令,也没有用户数据外发面——不读任何文件内容、输出只有安装状态文本(且照常过 output scrub)。`status` 的探测方式与 Agents 面板同源(本机 PATH 探测 + `--version`,数据不出本机)。
- **卸载的额外边界**:只支持 npm/brew 管理的安装,且带归属校验(不是该管理器装的不卸,防误删手动安装);script(curl|bash)装的没有反向操作,工具直接拒绝并引导按官方文档手动卸载;卸载只移除程序本体,不碰配置文件与历史数据。面板侧「卸载」按钮另有确认对话框;chat 侧沿用「用户明确要求」约束,无授权卡片。
- **触发前提**:system prompt 硬约束「仅当用户明确要求时才 install/update/uninstall」;`install`/`update`/`uninstall` 只是把任务交给后台引擎后立即返回,完成与否由前端 toast 通知,动作本身落本地 App 日志(`~/.kaya/logs/`,含 agent、动作、渲染出的完整命令)。
- **安装输出行先脱敏(R3,2026-08-31)**:子进程 stdout/stderr 的每一行进尾部缓冲与前端事件前,先过与工具输出同级的 secret scrub(install hook 输出不可控,可能携带 token / 带凭据 URL)。

---

## 8. 本地数据存储

**所有数据都在用户机器上**,无云同步。

| 数据 | 位置 | 谁写 | 保留期 |
|---|---|---|---|
| UI 状态(消息 / 设置) | `localStorage` key `kaya` | 前端 Zustand persist | 用户主动清 |
| LLM 上下文 | `~/.kaya/conversations/c_*.json` | Rust `ConversationStore` | 用户主动清 |
| 审计日志 | `~/.kaya/audit/s_*/events-*.jsonl` | 前端 `recordEvent` (Rust 调 fs) | 用户主动清 |
| Debug capture(可选) | `~/.kaya/debug/s_*/turn-N.json` | Rust debug module | dev-only,默认关 |
| App 诊断日志 | `~/.kaya/logs/app-YYYY-MM-DD.jsonl` | Rust `app_log`(前端 `snapshotCursor` 光标诊断探针;2026-08-26 起渠道重连错误按变化去重落盘,tag `channel` —— Windows GUI 构建无 stderr,这是该平台唯一留痕) | 手动清理 7 天前(Settings → App 日志);文件 0o600;**内容仅诊断元数据**(DOM/CSS 状态、计数、字节长度;渠道连接错误文本含主机名/错误描述,URL query/fragment 已脱敏不落 ticket),不含 PTY 输出/键入/secret |
| 终端滚动历史 | `~/.kaya/terminal/<tabId>.log` | Rust scrollback append(round-89) | 10MB cap + truncate-to-75% 轮转;tab 关闭即删 |
| Per-tab shell 历史 | `~/.kaya/history/<tab_id>.history` | Rust per-tab HISTFILE(round-111);Windows 移植 Phase 2 起 Windows 会话为 PSReadLine `HistorySavePath`(per-tab init.ps1 注入,等价 HISTFILE 机制) | tab 关闭 merge 回 `~/.zsh_history`;文件 0o600 |
| 登录令牌(refresh_token) | macOS 钥匙串(`com.kaya.auth` / dev `com.kaya.dev.auth`) | Rust `auth_tauri`(round-137/138,只写 refresh_token 单 item) | 登出 / 401 清理删除;access_token 不持久化(仅内存,15min TTL) |
| API Key(自备 key) | `localStorage` | 前端(不存磁盘文件) | 用户主动清;内置模型默认不使用 |
| Agents 供应商库(含 API Key) | `~/.kaya/agent-providers.json` | Rust agents 模块(Agents 管理中心「使用/保存」) | 用户主动删;**API Key 明文存储**(与 cc-switch 一致,设置 UI 已标注)。2026-08-20 review C3 起:**文件 0o600 原子写 + 读取时收紧旧版 0644 权限**(同机其它用户不可读);同时 scrub 正则覆盖 `"api_key": "…"` JSON 形态,LLM 即便经 trusted-read `cat` 读到该文件,key 值也会被 redact |
| IM 渠道会话映射 | `~/.kaya/channels/sessions.json` | Rust channels 模块(v2.1.0,IM 平台用户 ↔ 本机会话) | 用户主动清 |
| IM 镜像目标 chat | `~/.kaya/channels/mirror.json` | Rust channels 模块(Unreleased,镜像绑定跨重启恢复) | 用户主动清(渠道运行中删除不会在原地重建,绑定跨重启恢复随之失效,重启后回退 sole_key 启发式) |
| App 版本元信息(显示用) | 内存(`AppInfo` runtime 字段,**不持久化**) | Rust `get_app_info`(`src-tauri/src/app_info.rs`,round-56) | 进程内,重启重读 |

**App 版本元信息**(round-56, 2026-06-28):`get_app_info` Tauri 单例
命令从 `tauri::Builder` + `env!` 宏读 version / product_name /
git_commit / build_time / target triple / identifier,Settings → 「关于」
tab 和 Chat 空状态显示。**不写 localStorage、不写 audit JSONL、不写
磁盘** —— 纯内存,重启重读。详见
[`docs/superpowers/specs/2026-06-28-app-version-display-design.md`](./superpowers/specs/2026-06-28-app-version-display-design.md)。

**API Key** 只发到 LLM 鉴权 header,进 conversation JSON / audit JSONL
时会作为 scrub 类别处理。审计日志只记"用了哪个 provider",不记 key
内容。

**持久化前 sanitize**(`src/lib/messagesPersist.ts`):
- `reasoning` 字段**不存**到 localStorage — 思考链只在 audit JSONL
  里,刷新页面就丢(防止 localStorage 5MB 撑爆 + reasoning 太长无意义缓存)
- `toolCalls.outputFull` 不存(ephemeral "展开" copy,丢了可重跑命令)
- `toolCalls.output > 30KB` 截到 30KB 标 `outputTruncated: true`

`Message.kind` 字段保留(banner message 跨 reload 不丢,用户看到
"工具调用次数耗尽"提示不消失)。

---

## 9. 审计 Trail(事后取证)

源码: `src/lib/audit.ts` + `auditTypes.ts` + `auditEval.ts`

每次 chat session 开启时,前端 `recordEvent` 把事件落到
`~/.kaya/audit/s_<sessionId>/events-*.jsonl`(7 类定义,其中
`reasoning-chunk` 为 legacy,新会话实际落其余 6 类):

1. `session-start` — 会话开始(记 `previousSessionId`,链可追溯)
2. `user-input` — **scrub 后**的输入 + 原始字节数 + redactions
3. `tool-call-decided` — policy / safety 决策 + 阶段 + 理由
4. `tool-call-completed` — 执行结果 + redactions + error
5. `assistant-turn` — 完整一轮输出 + `systemPromptFingerprint` + `durationMs`
6. `reasoning-chunk` — 推理流式片段(**legacy**:round-114 起 reasoning 批量
   并入 `assistant-turn.reasoning` 字段,本事件仅为回放旧 JSONL 兼容保留)
7. `error` — 错误事件

`UserInputEvent.text` 是 scrub 后的文本,**不是**原始输入 — JSONL
文件本身不会泄漏 secret。

### 9.1 三层评估

Settings → Audit → "Evaluate" 对保存的会话跑三遍:

**Pass 1 · 规则回放** — 用当前版本的 policy + safety + scrub 规则,
对 recorded `tool-call-decided` 事件重跑。**任何决策 / 阶段 / 理由
的漂移**都会标记 — 规则升级后,过去哪些决策在今天会不一样,立刻可见。

**Pass 2 · 异常启发式**(4 条规则):
- `reader-without-redaction`:`cat` / `head` 等读取类跑了但 scrubber
  没命中(显式 trusted_read 阶段跳过)
- `network-without-redaction`:`curl` 输出 > 200B 但无 redaction
- `env-without-redaction`:`env` 输出含 `KEY=val` 但无 redaction
- `tool-call-without-reasoning-reflection`:thinking 模式开了,但 60s
  窗口内无 reasoning chunk(说明 LLM 发命令前没反思)

**Pass 3 · LLM-as-judge** — 唯一仍用 Vercel AI SDK v6 `streamText`
的代码路径(主 chat 走 Rust)。Judge 永远走 thinking mode —
审计是合规关键行为,不是用户偏好。Judge 自己的 COT 被捕获到
`JudgeResult.cot`,可在 UI 展开查看"评判的为什么"。

### 9.2 为什么需要三层

普通 LLM Agent 的失败模式是"闷声执行,事后才发现错了"。kaya
的设计哲学:你应当能**完整回放** AI 做的每一件事,包括**为什么**
做这件事。审计日志 + 三层评估 = 完整的"事后取证"能力,让"AI 是不是
把我的密钥传出去了"这种怀疑有可验证的答案。

---

## 10. Ollama 路径:零外发请求

Settings → Provider 选 **Ollama**,启动 `ollama serve`,完全本地推理:

- 推理不离开你的机器
- 无 API Key 需求
- 审计 / conversation / debug 仍按本机规则落盘
- 适用场景:隐私敏感工作、离线开发

---

## 11. 测试覆盖

### 11.1 Rust 端(`cargo test --lib`)

| 文件 | 测试数 | 锁的不变量 |
|---|---|---|
| `crates/kaya-core/src/llm/scrub.rs` | 106 | 30+ 类 secret 正则 + 整块 PEM 匹配 + 三档严格度映射;2026-08-20 review C3:JSON 形态 `"api_key": "…"` 值引号识别(`scrubs_api_key_json_quoted_value`) |
| `crates/kaya-core/src/llm/policy.rs` | 195 | 3 语义表 + 永久 blacklist + allowlist-miss + 管道分段 + trusted read + trusted download(round-57)+ `..`/前缀相似 escape;2026-08-20 review C1/C2:flag 内联值(`--name=value` / `-eSCRIPT`)+ 管道位显式路径降级 + tr/cut/sort/uniq 误杀回归 |
| `crates/kaya-core/src/llm/safety.rs` | 122 | DANGEROUS/REPL/OPERATION + heredoc/$(...)/`...` 递归 |
| `crates/kaya-core/src/llm/user_input_guard.rs` | 12 | backend 跑同一 pipeline,防 devtools 直 invoke 绕过前端 guard(round-71 +6) |
| `crates/kaya-core/src/agents/config.rs` | 16 | 读-改-写保留未知键 + hooks 键拒绝 + 0o600 原子写与旧 0644 收紧(2026-08-20 review C3) |
| `crates/kaya-core/src/llm/agent.rs` + `agent_loop_tests.rs` | 18 + 15 = 33 | abort flag RAII + stream-idle watchdog + debug response guard + run_chat_loop scripted harness(round-117) |
| `crates/kaya-core/src/llm/openai_compatible.rs` | 55 | preset dispatch + thinking_params per provider(含 deepseek-chat 排除) |
| `crates/kaya-core/src/llm/yunhou_chat.rs` | 48 | 内置 chat 消息转换 / 裁剪 / SSE 解析 / tool 聚合 / 错误映射 / mock 上游端到端(round-138) |
| `src-tauri/src/llm_commands.rs::tests` | 13 | wire format 锁:snake_case OK / camelCase MUST fail(防 "undefined" 误诊)+ `chat_args_accepts_builtin_chat_flag`(round-138) |

(2026-08-18 以 `grep -cE '#\[(tokio::)?test\]'` 按属性重数;该方法包含 `#[test]`、`#[tokio::test]` 及其他测试属性。)

> 数字是 2026-08-18 对上述模块的静态实测,不是估算。改 privacy 相关
> 代码后必须再数一遍并更新本节;不要用一次可能受已知 PTY 并行 flake
> 影响的 workspace 总数替代模块计数。

### 11.2 TS 端(`npm test`)

- `src/lib/__tests__/scrubber.test.ts` — TS mirror 测试,锁 PEM 整块、
  多 key 独立、public key 不被误判
- `src/lib/__tests__/runShellPolicy.test.ts` — policy TS mirror
- `src/lib/__tests__/safety.test.ts` — safety TS mirror
- `src/lib/__tests__/userInputGuard.test.ts` — `originalBytes` 反映原始
  输入、整块 PEM、emoji 不被切
- `src/lib/__tests__/audit.test.ts` / `auditEval.test.ts` — JSONL recorder
  + 3-pass evaluator

### 11.3 TS mirror ↔ Rust 同步(关键!)

`src/lib/{safety,runShellPolicy,scrubber}.ts` 是 Rust 端的 **deterministic
replay mirror**,**不是** live gate(实际 live gate 在 Rust 端)。存在的
理由:audit judge Pass 1 重放 recorded `tool-call-decided` 事件时用
TS mirror。**任何 Rust 端规则的修改必须同步改 TS mirror,否则 audit
judge 永远报 decision-drift**(已锁测试,见 `auditEval.test.ts`)。改时:

1. 先改 Rust 端(单测过)
2. 同步改 TS mirror
3. TS mirror 测过
4. 跑 auditEval 测过(Pass 1 漂移 0)
5. 更新 PRIVACY.md / README.md 涉及的部分

### 11.4 已知的高频漂移点

- `aws-access-key` / `github-pat-*` 这类 token 正则的字符类(`[A-Za-z0-9]`
  vs `[A-Za-z0-9_]` 漂移过两次,各锁了一个 round 的回归测试)
- PEM 类型变体(round-5 加 `ENCRYPTED`,round-7 加 `private key PRIVATE KEY`,
  每次都新加 BEGIN/END alternation pair)
- 命名映射(前端的 `strict` ≠ Rust 的 `Strict`,故意反着)
- placeholder 格式(round-19 统一到 `[REDACTED:<category>]`,前是
  Rust `[<category>]` / TS `[REDACTED:<category>]` 双格式)

---

## 12. 已知限制 / 不做

- **scrubber 是 regex,base64 编码过的 secret 拦不住** — 这是
  defense-in-depth 的"in-depth"含义:不指望它兜底,指望 policy 黑名单
  把这些命令先拒了
- **LLM 上下文在 LLM 端** — 我们 scrub 了发出去的内容,但 LLM 提供商
  自己的服务端日志 / 缓存策略我们管不着。选 Ollama 路径可彻底消除这一层
- **API Key 仍在 LLM 鉴权 header** — 这是协议设计,不可避免。但不进
  request body,审计日志也不记
- **trusted read 一旦放行就是整文件** — 你放一个 1MB log 进
  `~/.kaya/`,LLM 能 `cat` 整篇。scrubber 仍跑,但如果
  log 里有 1MB 随机数据,sed/grep 会把整篇带进 LLM context 撑爆 token
  预算 — 后果是任务失败,不是泄漏
- **用户在 terminal 里直接输入的命令不受任何门拦截** — 这是设计决策
  (用户的 shell 用户的责任),不是漏洞
- **Windows / Linux 不在 Phase 1** — 当前仅 macOS

---

## 13. 修改隐私相关代码时的检查清单

每改一处,都至少要:

- [ ] Rust 端 + TS mirror **同时改**(否则 audit judge Pass 1 报漂移)
- [ ] 涉及新 secret 类别:加 cargo 单测 + vitest 单测
- [ ] 涉及 scrub level 映射:`scrub.rs::frontend_name_to_scrub_level`
      头注释要更新
- [ ] 涉及 placeholder 格式:`scrub.rs` 和 `scrubber.ts` 两处 placeholder
      构造点同步
- [ ] 涉及用户输入 pipeline 改:`userInputGuard.ts` (前端) +
      `user_input_guard.rs` (Rust) + `input_guard.rs` (CLI) 三处同步
- [ ] 改完跑 `npm test && cargo test --workspace` 全绿
- [ ] 更新 `docs/PRIVACY.md` 涉及的部分(如果改了"已知限制"或"测试覆盖",
      文档要反映)

---

## 14. 一句话总结

> kaya **运行在你的 Mac** 上,**默认** LLM 看不到文件内容,
> 你说的每句话 / 工具的每个输出在出门前都被 5 道关卡过了一遍,所有
> 状态都留在本机可被回放审计。**Privacy First 是结构性的,不是
> 配置项 — 改不了**。
