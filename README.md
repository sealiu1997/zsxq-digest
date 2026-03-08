# zsxq-digest

## 简介 / Overview

`zsxq-digest` 是一个 OpenClaw skill，用来把 **知识星球（ZSXQ / zsxq.com）** 的最近更新整理成可读性更强的 digest。

`zsxq-digest` is an OpenClaw skill that turns recent **Knowledge Planet (ZSXQ / zsxq.com)** updates into readable digests.

它支持两类输出：
- 简短的日常分拣 digest
- 更适合阅读的分星球 stream digest

It supports two output styles:
- a short daily triage digest
- a richer per-circle stream digest

它主要帮助回答这些问题：
- 最近更新了什么？
- 哪些帖子值得优先点开？
- 每个星球最近都在讨论什么？

It helps answer questions like:
- what changed recently?
- which posts are worth opening first?
- what has each circle been discussing lately?

> 这是一个面向 **私有会员内容** 的 skill。
> 公开仓库只包含脚本、工作流和参考文档，**不会**包含你的 token、cookies、私有导出内容或本地运行态数据。

> This skill is designed for **private membership content**.
> The public repository only contains scripts, workflow, and references. It **does not** include your token, cookies, private exports, or local runtime state.

---

## MVP 方向 / MVP direction

首次授权方式：
- 引导式授权引导（guided auth bootstrap）

First-run authorization:
- guided auth bootstrap

稳定运行主模式：
- 本地私有 token / session 模式

Primary steady-state mode:
- local private token / session mode

次级恢复模式：
- 浏览器 relay 恢复与验证

Secondary recovery mode:
- browser relay recovery and verification

实验性备用路径：
- plain fetch fallback

Experimental fallback:
- plain fetch fallback

---

## 为什么是 token-first / Why token-first

很多 OpenClaw 用户会把 agent 跑在 VPS、Mac mini 或其他远程主机上，但真正登录知识星球的设备可能在别处。对这种部署方式来说，一个本地 gitignored 的 token 文件通常比长期依赖在线浏览器 relay 更稳。

Many OpenClaw users run agents on a VPS, Mac mini, or other remote hosts, while the actual ZSXQ login happens on another device. For that setup, a local gitignored token file is usually more practical than depending on a live browser relay all the time.

所以这个仓库明确区分了两件事：
- **bootstrap**：主机如何拿到第一个可复用的 session / token
- **steady state**：拿到 token 之后，后续 digest 如何稳定运行

So this repo explicitly separates:
- **bootstrap**: how the host gets its first reusable session / token
- **steady state**: how later digest runs work after that token already exists

---

## 安全说明 / Security warning

不要提交或发布以下内容：
- `state/session.token.json`
- 原始 cookies
- 含有私有认证信息的请求头
- 私有星球导出内容

Do **not** commit or publish:
- `state/session.token.json`
- raw cookies
- copied request headers containing private auth material
- private circle exports

推荐做法：
- 所有运行态数据都放在 gitignored 的 `state/` 下

Recommended rule:
- keep all runtime state under a gitignored `state/` directory

---

## 本地 token 文件 / Local token file

推荐路径：
- `state/session.token.json`

Recommended path:
- `state/session.token.json`

推荐结构：

Recommended schema:

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

## 快速开始 / Quick start

### 1. 列出已加入星球 / List joined circles

```bash
python3 scripts/collect_from_session.py \
  --token-file state/session.token.json \
  --mode groups
```

### 2. 抓取单个星球最近主题 / Fetch one circle's recent topics

```bash
python3 scripts/collect_from_session.py \
  --token-file state/session.token.json \
  --mode group-topics \
  --group-id <GROUP_ID> \
  --count 20 \
  --output /tmp/zsxq-items.json
```

### 3. 先导出本地 groups 配置 / Export a local groups config first

比较实用的做法是先从 live token 导出一个 `state/groups.json`，然后在本地把不想追踪的星球关掉。

A practical workflow is to export a starter `state/groups.json` from the live token, then locally disable circles you do not want in the digest.

```bash
python3 scripts/export_groups_config.py \
  --token-file state/session.token.json \
  --output state/groups.json
```

你也可以在导出时直接预先禁用某些星球。

You can also pre-disable circles while exporting.

```bash
python3 scripts/export_groups_config.py \
  --token-file state/session.token.json \
  --output state/groups.json \
  --disable-group-id <GROUP_ID>
```

### 4. 一次抓取多个星球 / Fetch multiple circles in one run

```bash
python3 scripts/collect_from_session.py \
  --token-file state/session.token.json \
  --mode multi-group-topics \
  --group-id <GROUP_ID_A> \
  --group-id <GROUP_ID_B> \
  --exclude-group-id <GROUP_ID_TO_SKIP> \
  --output /tmp/zsxq-multi.json
```

你也可以用 `groups.json` 驱动抓取，这样 disabled / ignored 的星球会在采集前就被跳过。

You can also drive collection from a `groups.json` file so disabled / ignored circles are skipped before collection.

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

### 5. 用 cursor 去重 / Deduplicate with a cursor file

```bash
python3 scripts/dedupe_and_state.py \
  /tmp/zsxq-multi.json \
  --cursor state/cursor.json \
  --access-mode token \
  --write-new-items /tmp/zsxq-new.json
```

### 6. 生成旧版简要 digest / Render the legacy digest

```bash
python3 scripts/digest_updates.py /tmp/zsxq-new.json
```

### 7. 生成新版 stream digest（推荐） / Render the new stream digest (recommended)

先对标准化后的条目做 enrich，得到分星球排序和布局字段。

First enrich normalized items for per-circle scoring and layout.

```bash
python3 scripts/enrich_stream_items.py \
  /tmp/zsxq-new.json \
  --output /tmp/zsxq-stream.json
```

然后渲染 markdown。

Then render markdown.

```bash
python3 scripts/render_stream_digest.py \
  /tmp/zsxq-stream.json
```

### 8. 一条命令跑旧版流水线 / Run the full legacy pipeline in one command

```bash
python3 scripts/run_digest_pipeline.py \
  --source token \
  --token-file state/session.token.json \
  --group-id <GROUP_ID_A> \
  --group-id <GROUP_ID_B> \
  --cursor state/cursor.json \
  --print-digest
```

### 9. 一条命令跑新版 stream 流水线（推荐） / Run the full stream pipeline in one command (recommended)

```bash
python3 scripts/run_stream_pipeline.py \
  --source token \
  --token-file state/session.token.json \
  --groups-file state/groups.json \
  --cursor state/cursor.json \
  --print-digest
```

### 9.1 私有增强模式 / Private enhancement mode

如果某些星球经常发“新闻串 / 链接串”这类内容，可以启用一个 **private enhancement mode**。它不会改变 public 默认逻辑，但会把新闻串类条目按“清洗后的原文线索”直接渲染出来，而不是再做一层抽象总结。

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

设计原则：
- public 默认模式保持中性、通用
- private mode 允许你针对本地工作流做更贴合的渲染

Design intent:
- the public default stays neutral and generic
- private mode can be tuned for your own local workflow without changing the public contract

### 10. 从浏览器抓取结果回放 / Browser fallback from a captured JSON snapshot

```bash
python3 scripts/collect_from_browser.py browser-capture.json --output /tmp/zsxq-browser.json
```

### 11. 浏览器恢复路径 / Browser-tool recovery path

当 token mode 失败时：
1. 用 OpenClaw browser tooling 打开一个已登录的知识星球页面
2. 用 `browser.snapshot` 确认当前页面确实能看到 feed
3. 通过 `scripts/browser_capture_starter.js` 配合 `browser.act(... evaluate ... )` 导出 JSON
4. 保存为 `browser-capture.json`
5. 再走正常的 browser-file 流水线

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

### 12. 统一浏览器授权工作流 / Unified browser bootstrap workflow

如果是第一次授权，推荐直接用统一入口。

For first-run browser-assisted authorization, use the single unified entrypoint.

第 1 步：准备 / 探测二维码状态，并更新 `state/auth-bootstrap.json`。

Step 1: prepare / probe QR state and update `state/auth-bootstrap.json`.

```bash
python3 scripts/run_browser_bootstrap.py run \
  --zsxq-ws-url ws://127.0.0.1:18800/devtools/page/ZSXQ \
  --wechat-ws-url ws://127.0.0.1:18800/devtools/page/WECHAT
```

这一阶段常见结果：
- `QR_READY` → 可以把二维码发给用户
- `AUTH_WAITING_CONFIRMATION` → 用户已扫码，等待确认
- `QR_EXPIRED` → 需要刷新二维码
- `AUTH_CAPTURE_UNVERIFIED` → 二维码流程已结束，下一步尝试 cookie capture

Typical result at this stage:
- `QR_READY` → send the QR to the user
- `AUTH_WAITING_CONFIRMATION` → user scanned, wait for confirmation
- `QR_EXPIRED` → refresh and resend
- `AUTH_CAPTURE_UNVERIFIED` → QR flow is gone; try cookie capture next

第 2 步：扫码/确认后，复用同一个入口，让它抓取 cookies、写入 `state/session.token.json` 并验证 token mode。

Step 2: after scan / confirmation, reuse the same entrypoint to capture cookies, finalize `state/session.token.json`, and verify token mode.

```bash
python3 scripts/run_browser_bootstrap.py run \
  --zsxq-ws-url ws://127.0.0.1:18800/devtools/page/ZSXQ \
  --wechat-ws-url ws://127.0.0.1:18800/devtools/page/WECHAT \
  --capture-ws-url ws://127.0.0.1:18800/devtools/page/LOGGED_IN_ZSXQ \
  --auto-finalize
```

默认会写出这些文件：
- `state/auth-bootstrap.json`
- `state/captured-cookies.json`
- `state/session.token.json`
- `state/groups.probe.json`

Default outputs:
- `state/auth-bootstrap.json`
- `state/captured-cookies.json`
- `state/session.token.json`
- `state/groups.probe.json`

如果 token 已经成功写入，但紧接着的验证请求碰到瞬时 API 故障，工作流仍然会返回 `status: ok`，并保留一个 `verification_warning` 字段，而不会假装验证完全成功。

If token capture succeeds but the immediate verification query hits a transient API-side failure, the workflow still returns `status: ok` and preserves a `verification_warning` field instead of silently pretending verification passed.

如果你想分阶段执行，下面这些底层 helper 仍然可以单独使用：
- `scripts/auth_bootstrap.py`
- `scripts/sync_auth_bootstrap.py`
- `scripts/capture_browser_cookies.js`
- `scripts/prepare_zsxq_qr_bootstrap.js`
- `scripts/probe_wechat_qr_state.js`

If you want to run the phases separately, these lower-level helpers still exist:
- `scripts/auth_bootstrap.py`
- `scripts/sync_auth_bootstrap.py`
- `scripts/capture_browser_cookies.js`
- `scripts/prepare_zsxq_qr_bootstrap.js`
- `scripts/probe_wechat_qr_state.js`

---

## 已知限制 / Known limitations

- token-mode HTTP 仍可能遇到 anti-bot 或 WAF 干扰
- 某些返回结构可能随客户端或站点版本变化
- 当前 MVP 保持轻量，不会激进展开长文
- token mode 失败时，浏览器恢复仍然是主恢复路径
- 当前 browser script 是“抓取结果标准化层”，不是完整浏览器控制器
- 如果 joined-group 列表里混入你不想跟踪的星球，优先用 `--groups-file` 或 `--exclude-group-id` 控制范围
- token mode 可能碰到 `code=1059: 内部错误` 这类瞬时 API 故障；当前 collector 会先重试再判失败
- private news-list rendering **不是** public 默认模式

- token-mode HTTP may still hit anti-bot or WAF behavior
- some response shapes may differ across clients or versions
- MVP keeps previews lightweight and does not aggressively expand long posts
- browser relay is still the main recovery path when token-mode fails
- the current browser script is a normalization layer for captured JSON, not a full browser controller by itself
- if joined-group listings contain circles you do not want, prefer `--groups-file` or `--exclude-group-id`
- token mode may hit intermittent API-side failures such as `code=1059: 内部错误`; the collector retries transient failures before treating the run as failed
- private news-list rendering is intentionally **not** the public default

---

## 显式错误状态 / Expected error statuses

- `TOKEN_MISSING`
- `TOKEN_INVALID`
- `TOKEN_EXPIRED`
- `ACCESS_DENIED`
- `EMPTY_RESULT`
- `QUERY_FAILED`
- `PARTIAL_CAPTURE`

---

## 授权引导与浏览器恢复 / Auth bootstrap and browser recovery

相关文档：
- `references/auth-bootstrap.md`
- `references/browser-recovery.md`
- `references/browser-workflow.md`

References:
- `references/auth-bootstrap.md`
- `references/browser-recovery.md`
- `references/browser-workflow.md`

当前 public MVP **不会**声称自己已经实现了一个完全验证过的 OAuth 式远程绑定流程。

The current public MVP does **not** claim a fully proven OAuth-style remote binding flow.

当前的决策逻辑是：
- 如果本地已经有可复用 token，就优先走 token mode
- 如果没有，就进入 guided browser bootstrap / recovery
- 如果环境里无法自动恢复出可复用 token，就退回 manual token import

Instead, it uses this decision model:
- if a reusable token already exists, use token mode
- if not, try a guided browser bootstrap / recovery path
- if the environment cannot recover a reusable token automatically, fall back to manual token import

当前 public MVP 也刻意把 browser mode 保持得比较轻：
- browser tooling 抓取可见 feed card
- 生成本地 JSON snapshot
- `scripts/collect_from_browser.py` 对 JSON 做标准化
- 然后回到正常 digest 流水线

The current public MVP also keeps browser mode intentionally lightweight:
- browser tooling captures visible feed cards
- a local JSON snapshot is produced
- `scripts/collect_from_browser.py` normalizes that snapshot
- the normal digest pipeline continues from there

---

## public 默认模式 vs 私有增强模式 / Public default vs private enhancement

### Public default

public / 可复用默认模式保持保守：
- 中性 stream digest
- 轻量组织
- 最少的可见元数据
- 普通讨论 / Q&A 走一句话摘要
- 不把圈子特化逻辑写死进 public contract

The public / reusable default stays conservative:
- neutral stream digest
- light organization
- minimal visible metadata
- one-sentence summaries for normal discussion / Q&A posts
- no circle-specific editorial logic baked into the public contract

### Private enhancement

本地私用时，你可以给特定星球打开定制渲染规则。

For local/private use, you may enable custom rendering rules for specific circles.

当前示例：
- `--private-news-list-mode`
- 如果一条帖子本身就是压缩过的新闻串 / 链接串，就直接保留清洗后的原始线索，而不是再强行总结一层

Current example:
- `--private-news-list-mode`
- if a post is already a compact news / link bundle, keep cleaned copied lines directly instead of forcing another summary layer

这样既能保持 public skill 可发布，又能允许你在本地工作流里做更贴合的增强。

This keeps the public skill publishable while still allowing higher-fit local use.

---

## 仓库建议边界 / Suggested repo boundary

建议公开仓库只包含：
- `SKILL.md`
- `README.md`
- `scripts/`
- `references/`
- `.gitignore`

Recommended public repo contents:
- `SKILL.md`
- `README.md`
- `scripts/`
- `references/`
- `.gitignore`

不要发布：
- `state/`
- token / session 文件
- 私有抓取结果或导出文件
- 带私人痕迹的 circle 名称或示例

Do not publish:
- `state/`
- personal token / session files
- private captures or exports
- personal circle names or private examples

---

## 打包说明 / Packaging note

公开发布时建议：
- 保持 skill package 精简
- 私有运行态数据不要进 git，也不要进 package
- 对外文档尽量都放在 skill 目录内，保持仓库自包含

For the final public repo:
- keep the skill package lean
- keep private runtime state out of git and out of the package
- keep public-facing docs inside the skill folder so the repo stays self-contained

如果 skill 目录里有本地 `state/` 文件，**不要**直接对 live working tree 用原始打包器。请用这个经过清洗的 helper：

If you have local `state/` files under the skill folder, **do not** package directly from the live working tree with the raw packager. Use the sanitized helper instead:

```bash
python3 scripts/package_public_release.py --output-dir ./dist
```

它会先生成一个干净 staging 副本，移除 `state/`、缓存文件和 `.bak-*`，然后再验证最终 `.skill` 包里不含私有运行态文件。

It stages a clean copy, strips `state/`, cache files, and `.bak-*`, then verifies that the final `.skill` package does not contain private runtime files.
