# zsxq-digest

## 简介 / Overview

`zsxq-digest` 是一个 OpenClaw skill，用来把 **知识星球（ZSXQ / zsxq.com）** 的最近更新整理成：
- 简短的日常分拣 digest
- 或更适合阅读的分星球 stream digest

它主要解决这几个问题：
- 最近更新了什么
- 哪些帖子值得优先点开
- 每个星球最近都在讨论什么

This skill is designed for **private membership content**. 公开仓库只包含工作流、脚本和参考文档，**不会**包含你的 token、cookies、私有导出内容或本地运行态数据。

---

## MVP 方向 / MVP direction

首次授权：
- **guided auth bootstrap / 引导式授权引导**

稳定运行主模式：
- **local private session / token / 本地私有 token 模式**

次级恢复模式：
- **browser relay recovery / verification / 浏览器恢复与验证**

实验性：
- plain fetch fallback

---

## 为什么是 token-first / Why token-first

很多 OpenClaw 用户会把 agent 跑在 VPS、Mac mini 或其他远程主机上，但真正登录知识星球的设备可能是另一台机器。对这类场景来说，使用一个本地 gitignored 的 token 文件，通常比长期依赖在线浏览器 relay 更稳。

So this repo explicitly distinguishes between:
- **bootstrap**: how the host gets the first reusable session / token
- **steady state**: how later digest runs work after that token already exists

---

## 安全说明 / Security warning

Do **not** commit or publish:
- `state/session.token.json`
- raw cookies
- copied request headers containing private auth material
- private planet exports

推荐规则：
- all runtime state should live under gitignored `state/`

---

## 本地 token 文件 / Local token file

推荐路径：
- `state/session.token.json`

推荐结构：

```json
{
  "kind": "cookie",
  "cookie_name": "zsxq_access_token",
  "cookie_value": "<private>",
  "domain": ".zsxq.com",
  "source": "browser-devtools-copy",
  "captured_at": "2026-03-08T14:30:00+08:00",
  "user_agent": "optional",
  "note": "stored locally only; do not commit"
}
```

---

## Quick start / 快速开始

### 1. List joined groups / 列出已加入星球

```bash
python3 scripts/collect_from_session.py \
  --token-file state/session.token.json \
  --mode groups
```

### 2. Fetch one group's recent topics / 抓取单个星球最近主题

```bash
python3 scripts/collect_from_session.py \
  --token-file state/session.token.json \
  --mode group-topics \
  --group-id <GROUP_ID> \
  --count 20 \
  --output /tmp/zsxq-items.json
```

### 3. Build a tracked groups config first / 先导出本地 groups 配置

A practical workflow is to export a starter `state/groups.json` from the live token, then locally disable circles you do not want in the digest.

```bash
python3 scripts/export_groups_config.py \
  --token-file state/session.token.json \
  --output state/groups.json
```

You can also pre-disable known ignored circles while exporting:

```bash
python3 scripts/export_groups_config.py \
  --token-file state/session.token.json \
  --output state/groups.json \
  --disable-group-id <GROUP_ID>
```

### 4. Fetch multiple groups in one run / 一次抓取多个星球

```bash
python3 scripts/collect_from_session.py \
  --token-file state/session.token.json \
  --mode multi-group-topics \
  --group-id <GROUP_ID_A> \
  --group-id <GROUP_ID_B> \
  --exclude-group-id <GROUP_ID_TO_SKIP> \
  --output /tmp/zsxq-multi.json
```

You can also drive this from a groups file so disabled / ignored circles are skipped before collection:

```json
[
  {"group_id": 10000000000001, "name": "Example Circle A", "enabled": true},
  {"group_id": 10000000000002, "name": "Example Circle B", "enabled": false, "status": "ignored"}
]
```

```bash
python3 scripts/collect_from_session.py \
  --token-file state/session.token.json \
  --mode multi-group-topics \
  --groups-file state/groups.json \
  --output /tmp/zsxq-multi.json
```

### 5. Deduplicate against bounded cursor state / 用 cursor 去重

```bash
python3 scripts/dedupe_and_state.py \
  /tmp/zsxq-multi.json \
  --cursor state/cursor.json \
  --access-mode token \
  --write-new-items /tmp/zsxq-new.json
```

### 6. Render a simple legacy digest / 生成旧版简要 digest

```bash
python3 scripts/digest_updates.py /tmp/zsxq-new.json
```

### 7. Render the new stream digest (recommended) / 生成新版 stream digest（推荐）

First enrich normalized items for per-circle scoring and layout:

```bash
python3 scripts/enrich_stream_items.py \
  /tmp/zsxq-new.json \
  --output /tmp/zsxq-stream.json
```

Then render markdown:

```bash
python3 scripts/render_stream_digest.py \
  /tmp/zsxq-stream.json
```

### 8. Run the full legacy pipeline / 一条命令跑旧版流水线

```bash
python3 scripts/run_digest_pipeline.py \
  --source token \
  --token-file state/session.token.json \
  --group-id <GROUP_ID_A> \
  --group-id <GROUP_ID_B> \
  --cursor state/cursor.json \
  --print-digest
```

### 9. Run the full stream pipeline (recommended) / 一条命令跑新版 stream 流水线（推荐）

```bash
python3 scripts/run_stream_pipeline.py \
  --source token \
  --token-file state/session.token.json \
  --groups-file state/groups.json \
  --cursor state/cursor.json \
  --print-digest
```

### 9.1 Private enhancement mode / 私有增强模式

For circles that often post grouped news / link-list updates, you can enable a **private enhancement mode**. It keeps the public default untouched, but renders those news-list entries as cleaned copied lines instead of forcing a second summarization layer.

```bash
python3 scripts/run_stream_pipeline.py \
  --source token \
  --token-file state/session.token.json \
  --group-id <GROUP_ID> \
  --cursor state/cursor.json \
  --private-news-list-mode \
  --print-digest
```

Design intent:
- public default remains neutral and generic
- private mode can be tuned for your own local workflow without changing the public contract

### 10. Browser fallback from a captured JSON snapshot / 从浏览器抓取结果回放

```bash
python3 scripts/collect_from_browser.py browser-capture.json --output /tmp/zsxq-browser.json
```

### 11. Browser-tool recovery path / 浏览器恢复路径

When token mode fails:
1. use OpenClaw browser tooling to open a logged-in Knowledge Planet page
2. confirm the visible feed with `browser.snapshot`
3. use the starter function in `scripts/browser_capture_starter.js` with `browser.act(... evaluate ... )`
4. save the returned JSON as `browser-capture.json`
5. run the normal browser-file pipeline

```bash
python3 scripts/run_stream_pipeline.py \
  --source browser-file \
  --browser-input browser-capture.json \
  --cursor state/cursor.json \
  --print-digest
```

### 12. Unified browser bootstrap workflow / 统一浏览器授权工作流

Use the single workflow entrypoint for first-run browser-assisted authorization.

Step 1: prepare / probe QR state and update `state/auth-bootstrap.json`:

```bash
python3 scripts/run_browser_bootstrap.py run \
  --zsxq-ws-url ws://127.0.0.1:18800/devtools/page/ZSXQ \
  --wechat-ws-url ws://127.0.0.1:18800/devtools/page/WECHAT
```

Typical result at this stage:
- `QR_READY` → send the QR to the user
- `AUTH_WAITING_CONFIRMATION` → user scanned, wait for confirmation
- `QR_EXPIRED` → refresh and resend
- `AUTH_CAPTURE_UNVERIFIED` → QR flow is gone; try cookie capture next

Step 2: after scan/confirmation, reuse the same entrypoint and let it capture cookies, finalize `state/session.token.json`, and verify token mode:

```bash
python3 scripts/run_browser_bootstrap.py run \
  --zsxq-ws-url ws://127.0.0.1:18800/devtools/page/ZSXQ \
  --wechat-ws-url ws://127.0.0.1:18800/devtools/page/WECHAT \
  --capture-ws-url ws://127.0.0.1:18800/devtools/page/LOGGED_IN_ZSXQ \
  --auto-finalize
```

Default outputs:
- `state/auth-bootstrap.json`
- `state/captured-cookies.json`
- `state/session.token.json`
- `state/groups.probe.json`

If token capture succeeds but the immediate verification query hits a transient API-side failure, the workflow still returns `status: ok` and preserves a `verification_warning` field instead of silently pretending verification passed.

If you want to run the phases separately, these lower-level helpers still exist:
- `scripts/auth_bootstrap.py`
- `scripts/sync_auth_bootstrap.py`
- `scripts/capture_browser_cookies.js`
- `scripts/prepare_zsxq_qr_bootstrap.js`
- `scripts/probe_wechat_qr_state.js`

---

## 已知限制 / Known limitations

- token-mode HTTP may still hit anti-bot or WAF behavior
- some response shapes may differ across clients or versions
- MVP keeps previews lightweight and does not aggressively expand long posts
- browser relay is still the main recovery path when token-mode fails
- current browser script is a normalization layer for captured JSON, not a full browser controller by itself
- joined-group listings may include circles you do not want in the digest; prefer `--groups-file` or `--exclude-group-id`
- token mode may hit intermittent API-side failures such as `code=1059: 内部错误`; current collector retries transient failures before treating the run as failed
- private news-list rendering is intentionally **not** the public default

---

## Expected error statuses / 显式错误状态

- `TOKEN_MISSING`
- `TOKEN_INVALID`
- `TOKEN_EXPIRED`
- `ACCESS_DENIED`
- `EMPTY_RESULT`
- `QUERY_FAILED`
- `PARTIAL_CAPTURE`

---

## Auth bootstrap and browser recovery

See:
- `references/auth-bootstrap.md`
- `references/browser-recovery.md`
- `references/browser-workflow.md`

The current public MVP does **not** claim a fully proven OAuth-style remote binding flow.
Instead it uses this decision model:
- if a reusable token already exists, use token mode
- if not, try a guided browser bootstrap / recovery path
- if the environment cannot recover a reusable token automatically, fall back to manual token import

The current public MVP also keeps browser mode intentionally lightweight:
- browser tooling captures visible feed cards
- a local JSON snapshot is produced
- `scripts/collect_from_browser.py` normalizes that snapshot
- the normal digest pipeline continues from there

---

## Public default vs private enhancement

### Public default
The public/reusable default stays conservative:
- neutral stream digest
- light organization
- minimal visible metadata
- one-sentence summaries for normal discussion / Q&A posts
- no circle-specific editorial logic baked into the public contract

### Private enhancement
For local/private use, you may enable custom rendering rules for specific circles.
Current example:
- `--private-news-list-mode`
- if a post is already a compact news/link bundle, keep cleaned copied lines directly instead of forcing another summary layer

This keeps the public skill publishable while still allowing higher-fit local use.

---

## Suggested repo boundary / 仓库建议边界

Recommended repo contents:
- `SKILL.md`
- `README.md`
- `scripts/`
- `references/`
- `.gitignore`

Do not publish:
- `state/`
- personal token/session files
- private captures or exports
- personal circle names or private examples

---

## Packaging note / 打包说明

For the final public repo:
- keep the skill package lean
- keep private runtime state out of the package and out of git
- keep public-facing docs inside this folder so the repo stays self-contained

If you have local `state/` files under the skill folder, do **not** package directly from the live working tree with the raw packager. Use the sanitized helper instead:

```bash
python3 scripts/package_public_release.py --output-dir ./dist
```

It stages a clean copy, strips `state/`, cache files, and `.bak-*`, then verifies the final `.skill` does not contain private runtime files.
