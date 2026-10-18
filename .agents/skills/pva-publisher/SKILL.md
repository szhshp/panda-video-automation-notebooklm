---
name: pva-publisher
description: Cross-platform video upload (login only when explicitly requested) — Bilibili, Douyin, Kuaishou, Weixin Video, YouTube. Also covers upload failures such as "Not logged in! Please run login script first" and the unreliable pva exit code.
---

# PVA Publisher

Cross-platform video upload using the `pva` CLI tool.

## Supported Platforms

| Platform | Alias | Login | Upload |
|----------|-------|-------|--------|
| Bilibili | `bilibili` | `npx pva bilibili login` | `npx pva bilibili upload` |
| Douyin | `douyin` | `npx pva douyin login` | `npx pva douyin upload` |
| Kuaishou | `kuaishou` | `npx pva kuaishou login` | `npx pva kuaishou upload` |
| Weixin Video | `weixin` / `wechat` / `weixinvideo` | `npx pva weixin login` | `npx pva weixin upload` |
| YouTube | `youtube` / `yt` | `npx pva youtube login` | `npx pva youtube upload` |

## Upload Options

- `--video <path>` — Path to video file (or `VIDEO_PATH` env)
- `--title <text>` — Video title (or `VIDEO_TITLE` env)
- `--desc <text>` — Video description (or `VIDEO_DESC` env). First line = headline title, at most **3 topics**, each line ≤ **50 chars**, **no emoji** (plain text only). Read it from `output/<slug>/spider/description.md` so every platform gets the same text.
- `--tags <list>` — Comma-separated tags, **max 3**. Most platforms restrict tags to 3. Pick the most relevant ones. (or `VIDEO_TAGS` env)
- `--cover <path>` — Cover image path (or `VIDEO_COVER` env)
- `--privacy <mode>` — YouTube only: `public|unlisted|private` (default: `unlisted`)
- `--headless` — Run browser in headless mode (default: **headed**)

## Running Multiple Platforms in Parallel

Login and upload tasks can run multiple platforms concurrently using shell backgrounding:

### Parallel Login

```bash
npx pva douyin login &
npx pva kuaishou login &
wait
```

### Parallel Upload Example

Artifacts live in the notebook's own folder — set `NB` to its slug first (see [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) → Step 0). Upload from **`output/$NB/`** (the upload-ready copy), never from `input/$NB/` (the working area).

```bash
cd "$(git rev-parse --show-toplevel)"   # pva needs absolute paths, so start from the project root
NB=zhipu-zcode

# Set the variables in a normal statement — do NOT chain them into the
# backgrounded command with `&&`. The subshells can start before the
# variables are assigned.
V="$(pwd)/output/$NB/video/video.mp4"
C="$(pwd)/output/$NB/video/cover.png"
T="$(jq -r .title "output/$NB/spider/title.json")"
G="$(jq -r '.tags | join(",")' "output/$NB/spider/tags.json")"
D="$(cat "output/$NB/spider/description.md")"

LOG="/tmp/pva_upload_${NB}_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$LOG"

for p in bilibili douyin kuaishou weixin; do
  npx pva "$p" upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
    > "$LOG/$p.log" 2>&1 &
done
wait

for p in bilibili douyin kuaishou weixin; do
  echo "--- $p ---"; tail -8 "$LOG/$p.log"
done
```

> ✅ 已验证可用。`jq` 直接读文件（不用管道），四平台后台并行跑，`wait` 等全部完成。

> ⚠️ **别只看整体退出码。** 每个平台各自打印自己的结论，而且不一样：bilibili / kuaishou / weixin 会断言成功提示；**douyin 从 pva v0.1.12 起改为等页面跳转到内容管理页（`/creator-micro/content/manage`），跳到了才打印 `Success: Douyin`**，60 秒没跳就退回 `Success: Douyin (check manually)`——只有看到这行才需要人工确认。所以每次都要逐个读 `/tmp/pva_upload_*/<平台>.log`。

### All headed (default)

```bash
npx pva douyin upload --video ./video.mp4 --title "Title" &
npx pva kuaishou upload --video ./video.mp4 --title "Title" &
wait
```

## Important Notes

- **Always use headed mode** (visible browser). Never use `--headless` for uploads.
- Each `pva` process launches its own Playwright browser instance — no conflict between platforms.
- Saved auth state is in `node_modules/@panda-video-automation/pva/playwright/.auth/` (local project dependency) — **which means a dependency reinstall deletes it silently.** See Pitfall 4.
- Login only needs to be done once; subsequent uploads reuse saved sessions — until the next reinstall.
- `npx pva --help` for full CLI reference.

## 🚨 Pitfalls & Solutions

### 1. `pva` command not found

PVA is a **local project dependency** (never global). The binary lives at `node_modules/.bin/pva` and is not on PATH — always invoke it via the local path or `npx`:

```bash
node_modules/.bin/pva bilibili upload ...
# NOT: pva bilibili upload
```

You can also use `npx`:
```bash
npx --no-install pva bilibili upload ...
```

### 2. Video/cover file path MUST be absolute

The `pva` CLI internally runs Playwright tests with `cwd: PROJECT_ROOT` (the npm package directory, e.g. `node_modules/@panda-video-automation/pva/`), NOT the project root. Relative paths like `output/<slug>/video/video.mp4` or `./output/<slug>/video/video.mp4` will fail because Playwright checks `existsSync()` from the wrong CWD.

**Always use absolute paths** for `--video` and `--cover`:

```bash
# ❌ Wrong — "Video file not found"
npx pva bilibili upload --video output/zhipu-zcode/video/video.mp4 ...

# ✅ Correct — use absolute path
npx pva bilibili upload --video /path/to/project/output/zhipu-zcode/video/video.mp4 ...

# ✅ Best — resolve in a bash variable
NB=zhipu-zcode
VIDEO="$(pwd)/output/$NB/video/video.mp4"
COVER="$(pwd)/output/$NB/video/cover.png"
npx pva bilibili upload --video "$VIDEO" --cover "$COVER" ...
```

### 3. Description with special characters / multi-line text

The `--desc` argument supports multi-line text, but shell escaping can be tricky with Chinese characters and newlines. Prefer reading from file into a variable:

```bash
NB=zhipu-zcode                                  # set the slug first — an unset $NB reads /description.md
DESC=$(cat "output/$NB/spider/description.md")
npx pva bilibili upload --desc "$DESC" ...
```

### 4. Auth state is wiped by a dependency reinstall

`pva` resolves login state as `path.join(process.cwd(), 'playwright/.auth', '<platform>.json')`, with `cwd` set to the package directory — so saved sessions live at `node_modules/@panda-video-automation/pva/playwright/.auth/`. Because that path is inside `node_modules`, **any reinstall removes it with no warning**: a `package.json` / `package-lock.json` change, `npm ci`, a fresh clone, a `node_modules` cleanup.

Upload then fails fast with:

```
Error: Not logged in! Please run login script first:
```

This is a login-judgement failure, not a flake and not a timeout. **Re-running the upload never fixes it** — the platform needs a fresh login. Confirm before launching anything:

```bash
ls -la node_modules/@panda-video-automation/pva/playwright/.auth/
```

An older copy of these files may survive at `~/.Trash/playwright/.auth/` (from when this project kept them at the repo root). Restoring one is a gamble: a Bilibili session still worked after ~4 months, while Douyin's and Kuaishou's at the same age were both rejected as not-logged-in. Treat restoring as a cheap first attempt only, and expect to re-login anyway.

### 5. Never close the browser during login

The login spec **deletes the existing auth file first**, then writes a new one only after login is detected:

```
🗑️  Deleting old authentication file: .../.auth/kuaishou.json
🌐 Opening Kuaishou...
Error: page.waitForTimeout: Target page, context or browser has been closed
```

Closing the window mid-login therefore leaves that platform with **no** auth file at all — strictly worse than the stale file it started with. Keep the browser open until the log prints `Authentication state saved`; the spec polls login status every 5 seconds and gives up after 15 minutes.

### 6. The CLI exit code is not a verdict — read the log

`pva` returns shell success even when the Playwright run inside it fails. An interrupted login returned `exit=0` to the caller while its own log ended in `1 failed`; a background task reported `completed` for the same run. Never branch on `$?` or on a background task's status — grep each platform's log for its own success line.

| Platform | Success signal | Trust |
|---|---|---|
| Bilibili | `Success: Bilibili` | ✅ asserts success |
| Kuaishou | `内容发布成功` | ✅ asserts success |
| Weixin | `Success: Weixin` | ✅ asserts success |
| Douyin | `Success: Douyin` | ✅ waits for redirect to `/creator-micro/content/manage` (pva ≥ v0.1.12) |
| Douyin | `Success: Douyin (check manually)` | ⚠️ publish clicked, redirect not observed within ~60s — **verify in the app before re-running**, or you double-post |

The `(check manually)` verdict is genuinely uncertain, not a soft failure: a re-run in the same situation landed the real `Success: Douyin`, meaning the first attempt had not published. Check the platform's content manager first, then decide.

## Behavioral Rules

- **Pre-flight the auth files; do not interrogate the user.** Before an upload run, list the auth directory and confirm every target platform has a `<platform>.json`. This is a file listing — not a login check, not a `--check` round-trip, and it launches nothing. It catches the wiped-auth case (Pitfall 4) before four browsers spin up.
- Do not ask "are you logged in?". A *missing* file is answered by the listing; a *present but stale* file is answered by the upload itself.
- **Never run `login` on your own initiative.** It is interactive (the user must scan a QR code), and it deletes the old auth file before saving a new one (Pitfall 5). When the listing shows missing auth, report which platforms are affected and ask the user to log in.
- When the user says "upload to [platforms]", directly open upload commands for all listed platforms in **parallel** (using `&` + `wait`).
- **Upload from `output/<slug>/`, never from `input/<slug>/`.** `input/` is the working area (clean master, watermarked master, versioned BGM mixes); `output/` is the upload-ready copy. Title: `output/<slug>/spider/title.json`. Video: `output/<slug>/video/video.mp4`. Cover: `output/<slug>/video/cover.png`. Never read from either root — `input/` root holds only the shared `watermark.png`.
- **Read every platform's log before reporting success.** The CLI exit code is not a verdict (Pitfall 6).
