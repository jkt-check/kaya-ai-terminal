# kaya · Privacy First Design

> **The top priority is user privacy. Never leak private information to the LLM.**
>
> This is the starting point for product decisions — not a "security note" tacked on at
> the end of the document. Before writing the first line of code, every design must answer
> the question: **"Once this data goes out, can it be recalled?"** If not, redact it
> before it leaves.

---

## 1. The One-Sentence Promise

> kaya runs on **your own Mac**, and the LLM only sees what you **actively give it**.
> **By default it cannot see any of your files.** Whatever you put under `~/.kaya/`,
> whatever secrets are in what you type — only after passing through 5 gates does it
> enter the LLM's context.

This is structural, not a configuration option — it cannot be changed.

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

**Only three things** leave your machine: the scrubbed user input, the scrubbed tool
output, and the scrubbed LLM request body. **The API key itself** is only sent in
the LLM auth header — never in the body. **Prompt content** is post-scrubbing content,
not the raw input.

**All state stays on the local machine**: UI state lives in `localStorage`; LLM
context / audit / debug live under `~/.kaya/`. State storage has no central
server, no telemetry, no cloud sync.

**Default model source's data egress (round-138, 2026-08-04)**: by default kaya
uses the **built-in model** (the yunhou `/chat` proxy) — scrubbed chat content
is sent over your login token (JWT) to the **yunhou first-party server**, which
answers with yunhou's DeepSeek key. Before sending, the client trims to the
server's limits (message count / per-message bytes / total bytes — the oldest
history is dropped first; dropping content before send is itself a
privacy-relevant behavior). The login token (refresh_token) lives only in the
macOS Keychain. Switching to **Ollama** (local inference) achieves zero egress;
switching to **OpenAI compatible** sends content to your chosen provider. Local
state storage still has no central server — but the default data **egress**
goes through the yunhou server, a change since round-138 that the privacy
positioning must state explicitly.

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
   │         read (cat ~/.kaya/*   │
   │         / head / tail / grep / awk /     │
   │         sed -n etc.) + trusted download  │
   │         (curl -o ~/.kaya/     │
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
   │  Failure mode: base64-encoded secrets    │
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

---

## 5. Gates 1 + 2: Policy + Safety Gates

LLM-invoked `run_shell` goes through the first two stages of the Rust 4-stage
pipeline:

### 5.1 Policy (capability policy, 3 semantic tables + 7 deny categories)

Source: `crates/kaya-core/src/llm/policy.rs` (280 cargo tests, re-measured 2026-08-31). Since round-67 (2026-07-05), the capability policy is **no longer a binary allow/deny-list structure** — it's **3 semantic tables (METADATA / STREAM_TRANSFORMERS / FILE_READERS) + a permanent blacklist + an allowlist-miss catch-all**, and the deny reason carries a structured `[stage:subcategory]` prefix (see §5.1.1). The system prompt (`policy_prompt::render_allowed_section_*`) and the run_shell tool description (`tools/run_shell.rs::rules_prose`) are generated from the same data source — single source of truth, no drift.

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
  inspect or delete at any time. See
  [`docs/superpowers/specs/2026-06-28-trusted-download-design.md`](./superpowers/specs/2026-06-28-trusted-download-design.md)
  for details.

- **3 semantic tables** (round-67, see §5.1.1): **METADATA** (`ls` / `ps` / `df` / `uname` etc., pure metadata, always allowed), **STREAM_TRANSFORMERS** (`head` / `sort` / `grep` / `awk` etc., always allowed in non-first pipeline positions; first position allowed with no file arg or a file under `~/.kaya/`), **FILE_READERS** (`cat` / `xxd` / `strings` / `base64` etc., only allowed when the file is under `~/.kaya/`). `cat` / `xxd` / `strings` / `base64` are **conditionally allowed** under `~/.kaya/` (trusted_read), not unconditionally blacklisted. **v2.1.10 review C1 tightening**: `source` was removed from FILE_READERS — `source <file>` *executes* the file rather than reading it, and combined with trusted download it formed a no-confirmation RCE chain; it now always falls to allowlist-miss deny (the `. file` form stays blocked by the dot-source blacklist).
- **Permanent blacklist** (never allowed, deny prefix `[policy:blacklist]`): shell escape / `eval` / `exec` / `xargs` / `curl` (outside trusted_download) / `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` / etc. **Attribution note**: `rm` / `sudo` are denied by the safety layer's DANGEROUS patterns (`[safety:dangerous]`), `osascript` by the safety layer's OPERATION patterns (`[safety:operation]`), and `tee` — after round-67 removed it from STREAM_TRANSFORMERS — falls into `[policy:allowlist-miss]`; none of these four live in the policy-layer permanent blacklist.
- **Pipeline-segment awareness**: In `du | sort | head`, `sort` / `head` in
  non-first positions are "pure byte-stream transformers" and **do not** trigger
  the content-read blacklist.
- **Trusted Read sub-stage** (2026-06-17, see `2026-06-17-trusted-read-root-design.md`):
  If the path args normalize to under `~/.kaya/`, allow `cat` / `head` / `tail` /
  `grep` / `awk` / `bat` / `view` / `sed -n` (writes are split by type:
  `sed -i` / `tee` / `mv` / `rm` are **still blocked entirely**; since
  round-68 (2026-08-05), `>` / `>>` redirects are allowed when the target
  normalizes to under `~/.kaya/` and has a safe extension (non-empty, not in
  BLOCKED_DOWNLOAD_EXTS), with `/dev/null` as an exception; `<>` is gated as
  a write, and `<` input redirects are gated by the trusted-read rules).
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
| **METADATA** | `ls` / `ps` / `netstat` / `df` / `wc` / `find` / `uname` / `whoami` / `date` etc. | always | never (exceptions since the v2.1.10 review: `wc` / `md5` / `shasum` / `sha256sum` with an out-of-root file path arg are denied as trusted-read-miss — content/size oracles; on the Windows flavor, args referencing `env:` / `$env:` are denied — env dump; since R2: `command` only allows the `-v/-V <name>` pure query, and the `rustup run` / `npm exec|dlx|run|run-script|explore|init|create` (same for pnpm) / non-query `yarn` (whitelist inversion — yarn classic's bare-script shorthand `yarn build` included, bare `yarn` means install) / `cargo run|build|test|bench|install|script` / `pip download` / `brew ruby|irb` subcommand exec surface is denied with [policy:blacklist]; `bun` is removed from the allowlist entirely (R5 — a JS runtime: `bun -e` / `bun --print` / `bun <file>` are arbitrary execution, aligned with node/python/deno → allowlist-miss)) |
| **STREAM_TRANSFORMERS** | `head` / `tail` / `sort` / `grep` / `awk` / `sed` / `cut` / `tr` / `uniq` etc. | non-first position always; first position with no file arg; first position + file under `~/.kaya/` | first position + file outside `~/.kaya/`; `sed -i` (a write) always; sed script-body `w` / `r` / `R` / `e` commands always (v2.1.10 review H1); path-less recursive searches (`grep -r`, bare `rg`) when cwd is outside the root (v2.1.10 review M-sec-2) |
| **FILE_READERS** | `cat` / `bat` / `view` / `less` / `more` / `xxd` / `hexdump` / `od` / `strings` / `base64` / `base32` (`source` removed in v2.1.10 review C1 — it executes rather than reads; always allowlist-miss) | file arg under `~/.kaya/` | file outside `~/.kaya/` |
| **Permanent blacklist** | shell escape / `eval` / `exec` / `xargs` / `curl` / `wget` / `nc` / `env` / `printenv` / `set` / `export` / `pbcopy` / `screencapture` / `open` / `history` etc. (`rm`/`sudo` belong to safety DANGEROUS, `osascript` to safety OPERATION, `tee` falls into allowlist-miss — none are in this table) | never | always |

**The 7 deny subcategories** (the `[stage:subcategory]` prefix):

- `[safety:dangerous]` destructive operations (`rm -rf` / `sudo` / `dd if=` / `mkfs` / `kill -9`)
- `[safety:repl]` REPL/interactive (`vim` / `less` / bare `python` / bare `node` / `irb` / `gh repl`)
- `[safety:operation]` system mutation (`npm install` / `brew install` / `systemctl` / `launchctl`)
- `[policy:blacklist]` permanent blacklist (shell escape / eval / exec / xargs / curl / wget / env dump / pbcopy / screencapture / open / history)
- `[policy:allowlist-miss]` base command not in any of the 3 allowlist tables
- `[policy:trusted-read-miss]` file reader / stream transformer with a path outside `~/.kaya/`
- `[policy:trusted-download-miss]` `curl -o` target outside `~/.kaya/` or unsafe extension (`.sh` / `.py` / `.exe` / `.dmg` / binary / installer)

**Privacy-surface analysis**: this change **does not widen the leak surface**. Legitimate stream-filter behavior like `ps aux | grep nginx` (which the old model had to special-case via "PIPELINE_CONSUMERS skip") is now first-class allowed; content reads (`cat` / `xxd` / `strings` / `base64`) are still gated on the file living under `~/.kaya/`, and the scrubber still runs on anything read back. The audit `ToolCallCompleted` event is stamped with the gate's stage so the audit judge's `reader-without-redaction` rule can skip legitimately-allowed content reads / downloads.

### 5.2 Safety (dangerous-command detection)

Source: `crates/kaya-core/src/llm/safety.rs` (test count tracked in §11.1, three
sets of regex)

- **DANGEROUS**: `rm -rf` / disk / direct device writes / `dd if=` / `mkfs` / etc.
- **REPL**: Bare invocations of `vim` / `less` / `python` / `node` / `irb` (which
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

---

## 7. Gate 4: Output Scrub (Credential Redaction)

Source: `crates/kaya-core/src/llm/scrub.rs` + TS mirror `src/lib/scrubber.ts`

### 7.1 Round-54 (2026-06-25) Three-Tier Design

The ~38 secret / PII patterns are divided into three tiers based on "must this
never leak?" (v2.1.10 review R2 added 8 categories against common gitleaks
rules — see the end of the Tier 0 list):

**Tier 0 — Core (runs at every level; Minimal / Balanced / Strict all enable it)**

Core secrets that must never be level-gated — even if the user picks "strict"
(the weakest UI option), API keys / PEM private keys cannot leak. ~28 categories:

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
  `placeholder` etc. are not accidentally matched)
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
[`docs/superpowers/specs/2026-06-25-scrub-default-aggressive-design.md` §11](./superpowers/specs/2026-06-25-scrub-default-aggressive-design.md).

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
  [`docs/superpowers/specs/2026-06-12-llm-secrets-redaction-design.md`](./superpowers/specs/2026-06-12-llm-secrets-redaction-design.md)).
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
output of a chosen tab (a per-tab 64 KiB ANSI-stripped ring buffer, in memory
only, never written to disk; while a full-screen TUI (vim, AI CLIs, etc.) is
running, capture switches to a live screen snapshot — the built-in terminal
emulator (vt100) reconstructs the screen in memory, so what is read is the
current screen text and cursor position, likewise in memory only and never
written to disk, and filtered through the same scrub pipeline as every tool
before reaching the AI).

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

**Interaction watch (2026-08-16, B)**: the watcher behind tab
waiting-for-confirmation reminders is **read-only** — it only pattern-matches
against screen snapshots and never injects input into the terminal; the
confirmation summary (excerpt) shown on the notification card is
secret-scrubbed and used solely for local display and notification — it is
never sent to the model. **Exception (coding mode, 2026-08-17)**: when you
explicitly enable coding mode, the scrubbed excerpt above is included in a
chat turn as part of the host instruction text — i.e. it is sent to your
selected model along with that turn, so kaya can proactively relay which tab
is waiting for your decision / which task has finished, and ask you for
instructions. Coding mode is off by default; turning it off restores the
"never sent to the model" behavior.

**Agent config file read/write (2026-08-17, P2-F1)**: chat's `agent_config`
tool can read and modify the CLI config files of the three supported coding
agents: claude-code → `~/.claude/settings.json`, kimi-code →
`~/.kimi-code/config.toml`, opencode → `~/.config/opencode/opencode.json`.
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
- **`hooks` keys are always refused**: that key can bind arbitrary shell
  commands to the agent's later runs; merge requests containing it are denied
  outright (no grant card is even shown) — edit it manually instead.
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
- **Install output lines are scrubbed first (R3, 2026-08-31)**: every line of
  the child process's stdout/stderr is run through the same secret scrub as
  tool outputs before entering the tail buffer and the frontend event stream
  (install-hook output is uncontrolled and may carry tokens / credential
  URLs).
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

---

## 8. Local Data Storage

**All data lives on the user's machine**; no cloud sync.

| Data | Location | Written by | Retention |
|---|---|---|---|
| UI state (messages / settings) | `localStorage` key `kaya` | Front-end Zustand persist | User clears manually |
| LLM context | `~/.kaya/conversations/c_*.json` | Rust `ConversationStore` | User clears manually |
| Audit log | `~/.kaya/audit/s_*/events-*.jsonl` | Front-end `recordEvent` (Rust invokes fs) | User clears manually |
| Debug capture (optional) | `~/.kaya/debug/s_*/turn-N.json` | Rust debug module | dev-only, off by default |
| App diagnostic log | `~/.kaya/logs/app-YYYY-MM-DD.jsonl` | Rust `app_log` (front-end `snapshotCursor` cursor diagnostic probes; since 2026-08-26 channel reconnect errors are deduped-on-change into this log, tag `channel` — Windows GUI builds have no stderr, this is the only durable trail on that platform) | manual prune of logs older than 7 days (Settings → App log); file 0o600; **content is diagnostic metadata only** (DOM/CSS state, counters, byte lengths; channel connection error text may contain hostnames/error descriptions, URL query/fragment redacted so no ticket is persisted), no PTY output / keystrokes / secrets |
| Terminal scrollback | `~/.kaya/terminal/<tabId>.log` | Rust scrollback append (round-89) | 10MB cap + truncate-to-75% rotation; deleted on tab close |
| Per-tab shell history | `~/.kaya/history/<tab_id>.history` | Rust per-tab HISTFILE (round-111) | merged back to `~/.zsh_history` on tab close; file 0o600 |
| Login token (refresh_token) | macOS Keychain (`com.kaya.auth` / dev `com.kaya.dev.auth`) | Rust `auth_tauri` (round-137/138, single refresh_token item) | deleted on logout / 401 cleanup; access_token not persisted (memory only, 15min TTL) |
| API Key (your own key) | `localStorage` | Front-end (not written to disk files) | User clears manually; unused by default built-in model |
| Agents provider library (incl. API keys) | `~/.kaya/agent-providers.json` | Rust agents module (Agents panel "Use"/"Save") | User deletes manually; **API keys stored in plaintext** (same as cc-switch; noted in the settings UI) |
| App version metadata (display) | Memory (`AppInfo` runtime field, **not persisted**) | Rust `get_app_info` (`src-tauri/src/app_info.rs`, round-56) | Within process; re-read on restart |

**App version metadata** (round-56, 2026-06-28): The `get_app_info` Tauri singleton
command reads version / product_name / git_commit / build_time / target triple /
identifier from `tauri::Builder` + `env!` macros, displayed by Settings → "About"
tab and the Chat empty state. **Not written to localStorage, not written to audit
JSONL, not written to disk** — purely in memory; re-read on restart. See
[`docs/superpowers/specs/2026-06-28-app-version-display-design.md`](./superpowers/specs/2026-06-28-app-version-display-design.md).

**API Key** is only sent in the LLM auth header. When it enters the conversation
JSON / audit JSONL it is processed as a scrub category. The audit log only
records "which provider was used", never the key contents.

**Sanitize before persistence** (`src/lib/messagesPersist.ts`):
- The `reasoning` field is **not stored** in localStorage — the chain of thought
  lives only in the audit JSONL, and is discarded on page reload (prevents the
  localStorage 5 MB cap from blowing up + reasoning is too long to be worth
  caching)
- `toolCalls.outputFull` is not stored (the ephemeral "expand" copy; can be
  regenerated by re-running the command)
- `toolCalls.output > 30KB` is truncated to 30 KB and tagged
  `outputTruncated: true`

The `Message.kind` field is preserved (banner messages survive reload — the
user keeps seeing the "tool-call budget exhausted" notice).

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

### 9.2 Why Three Layers Are Needed

The failure mode of ordinary LLM agents is "execute silently, and only notice
the mistake afterward". kaya's design philosophy: you should be able to **fully
replay** every action the AI took, including **why** it took that action.
Audit log + three-layer evaluation = a complete "post-hoc forensics" capability
that gives "did the AI send my secret out?" a verifiable answer.

---

## 10. Ollama Path: Zero Outbound Requests

Settings → Provider picks **Ollama**, start `ollama serve`, fully local inference:

- Inference never leaves your machine
- No API Key required
- Audit / conversation / debug still follow the local rules for disk writes
- Use cases: privacy-sensitive work, offline development

---

## 11. Test Coverage

### 11.1 Rust Side (`cargo test --lib`)

| File | Test count | Locked invariants |
|---|---|---|
| `crates/kaya-core/src/llm/scrub.rs` | 105 | 30+ secret regexes + whole PEM block matching + three-level strictness mapping |
| `crates/kaya-core/src/llm/policy.rs` | 280 | 3 semantic tables + permanent blacklist + allowlist-miss + pipeline segmentation + trusted read + trusted download (round-57) + `..`/prefix-similarity escapes |
| `crates/kaya-core/src/llm/safety.rs` | 122 | DANGEROUS/REPL/OPERATION + heredoc/$(...)/`...` recursion |
| `crates/kaya-core/src/llm/user_input_guard.rs` | 12 | backend runs the same pipeline; prevents devtools direct-invoke from bypassing the front-end guard (round-71 +6) |
| `crates/kaya-core/src/llm/agent.rs` + `agent_loop_tests.rs` | 18 + 15 = 33 | abort flag RAII + stream-idle watchdog + debug response guard + scripted `run_chat_loop` harness (round-117) |
| `crates/kaya-core/src/llm/openai_compatible.rs` | 55 | preset dispatch + thinking_params per provider (incl. deepseek-chat exclusion) |
| `crates/kaya-core/src/llm/yunhou_chat.rs` | 48 | builtin-chat message conversion / trimming / SSE parsing / tool aggregation / error mapping / mock-upstream end-to-end (round-138) |
| `src-tauri/src/llm_commands.rs::tests` | 13 | wire-format lock: snake_case OK / camelCase MUST fail (prevents "undefined" mis-diagnosis) + `chat_args_accepts_builtin_chat_flag` (round-138) |

(Counts re-measured on 2026-08-18 via `grep -cE '#\[(tokio::)?test\]'` per attribute, which includes all test attributes.)

> These numbers are actual measurements from 2026-08-18, not estimates. After
> changing privacy-related code you must re-run and update this section —
> drifting numbers indicate the test suite is degrading.

### 11.2 TS Side (`npm test`)

- `src/lib/__tests__/scrubber.test.ts` — TS mirror tests; locks whole-block PEM,
  multiple keys independent, public key not mis-flagged
- `src/lib/__tests__/runShellPolicy.test.ts` — policy TS mirror
- `src/lib/__tests__/safety.test.ts` — safety TS mirror
- `src/lib/__tests__/userInputGuard.test.ts` — `originalBytes` reflects raw
  input, whole-block PEM, emojis not cut
- `src/lib/__tests__/audit.test.ts` / `auditEval.test.ts` — JSONL recorder
  + 3-pass evaluator

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

- **The scrubber is a regex; base64-encoded secrets cannot be caught** — this
  is the "in-depth" part of defense-in-depth: don't rely on it as the last
  line, rely on the policy blacklist to reject those commands first.
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
- **Windows / Linux are not in Phase 1** — currently macOS only.

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
- [ ] After changes, run `npm test && cargo test --workspace` — all green
- [ ] Update the relevant parts of `docs/PRIVACY.md` (if "Known Limitations"
      or "Test Coverage" changed, the doc must reflect it)

---

## 14. One-Sentence Summary

> kaya **runs on your Mac**, and **by default** the LLM cannot see file
> contents. Every word you say and every tool output goes through 5 gates
> before leaving the device, and all state stays on the local machine where
> it can be replayed and audited. **Privacy First is structural, not a
> configuration option — it cannot be changed.**