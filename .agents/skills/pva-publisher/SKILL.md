---
name: pva-publisher
description: Cross-platform video upload (login only when explicitly requested) — Bilibili, Douyin, Kuaishou, Weixin Video, YouTube
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
- `--desc <text>` — Video description (or `VIDEO_DESC` env). First line = headline title, at most **3 topics**, each line ≤ **30 chars**, **no emoji** (plain text only).
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

```bash
cd /{PROJECT_ROOT}/panda-video-automation-notebooklm
V="$(pwd)/input/video.mp4" && C="$(pwd)/input/cover.png" \
  && T="$(jq -r .title input/title.json)" \
  && G="$(jq -r '.tags | join(",")' input/tags.json)" \
  && D="$(cat input/description.md)"

npx pva bilibili upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" &
npx pva douyin upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" &
npx pva kuaishou upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" &
npx pva weixin upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" &
wait
```

> ✅ 这条命令已验证可用。直接用 `jq` 读文件（不用管道），`npx pva` 确保能找到命令，四平台后台并行跑，`wait` 等全部完成。

### All headed (default)

```bash
npx pva douyin upload --video ./video.mp4 --title "Title" &
npx pva kuaishou upload --video ./video.mp4 --title "Title" &
wait
```

## Important Notes

- **Always use headed mode** (visible browser). Never use `--headless` for uploads.
- Each `pva` process launches its own Playwright browser instance — no conflict between platforms.
- Saved auth state is in `node_modules/@panda-video-automation/pva/playwright/.auth/` (local project dependency).
- Login only needs to be done once; subsequent uploads reuse saved sessions.
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

The `pva` CLI internally runs Playwright tests with `cwd: PROJECT_ROOT` (the npm package directory, e.g. `node_modules/@panda-video-automation/pva/`), NOT the project root. Relative paths like `input/video.mp4` or `./input/video.mp4` will fail because Playwright checks `existsSync()` from the wrong CWD.

**Always use absolute paths** for `--video` and `--cover`:

```bash
# ❌ Wrong — "Video file not found"
npx pva bilibili upload --video input/video.mp4 ...

# ✅ Correct — use absolute path
npx pva bilibili upload --video /path/to/project/input/video.mp4 ...

# ✅ Best — resolve in a bash variable
VIDEO="$(pwd)/input/video.mp4"
COVER="$(pwd)/input/cover.png"
npx pva bilibili upload --video "$VIDEO" --cover "$COVER" ...
```

### 3. Description with special characters / multi-line text

The `--desc` argument supports multi-line text, but shell escaping can be tricky with Chinese characters and newlines. Prefer reading from file into a variable:

```bash
DESC=$(cat input/description.md)
npx pva bilibili upload --desc "$DESC" ...
```

## Behavioral Rules

- **Do NOT check login status unless the user explicitly mentions it.** Skip `--check` / login verification steps entirely by default.
- When the user says "upload to [platforms]", directly open upload commands for all listed platforms in **parallel** (using `&` + `wait`).
- Do not ask "are you logged in?" or attempt to verify credentials upfront. If a platform requires login, the browser will prompt for it naturally.
- For the **input folder**: read `input/title.json` for the title, use `input/*.mp4` for the video path.
