---
name: pva-publisher
description: Cross-platform video upload (login only when explicitly requested) — Bilibili, Douyin, Kuaishou, Weixin Video, YouTube
---

# PVA Publisher

Cross-platform video upload using the `pva` CLI tool.

## Supported Platforms

| Platform | Alias | Login | Upload |
|----------|-------|-------|--------|
| Bilibili | `bilibili` | `pva bilibili login` | `pva bilibili upload` |
| Douyin | `douyin` | `pva douyin login` | `pva douyin upload` |
| Kuaishou | `kuaishou` | `pva kuaishou login` | `pva kuaishou upload` |
| Weixin Video | `weixin` / `wechat` / `weixinvideo` | `pva weixin login` | `pva weixin upload` |
| YouTube | `youtube` / `yt` | `pva youtube login` | `pva youtube upload` |

## Upload Options

- `--video <path>` — Path to video file (or `VIDEO_PATH` env)
- `--title <text>` — Video title (or `VIDEO_TITLE` env)
- `--desc <text>` — Video description (or `VIDEO_DESC` env). First line = headline title, at most **3 topics**, each line ≤ **30 chars**.
- `--tags <list>` — Comma-separated tags, **max 3**. Most platforms restrict tags to 3. Pick the most relevant ones. (or `VIDEO_TAGS` env)
- `--cover <path>` — Cover image path (or `VIDEO_COVER` env)
- `--privacy <mode>` — YouTube only: `public|unlisted|private` (default: `unlisted`)
- `--headless` — Run browser in headless mode (default: **headed**)

## Running Multiple Platforms in Parallel

Login and upload tasks can run multiple platforms concurrently using shell backgrounding:

### Parallel Login

```bash
pva douyin login &
pva kuaishou login &
wait
```

### Parallel Upload (one-liner, all 4 platforms)

```bash
V="$(pwd)/input/video.mp4" && C="$(pwd)/input/cover.png" \
  && T="$(cat input/title.json | jq -r .title)" \
  && G="$(cat input/tags.json | jq -r '.tags | join(",")')" \
  && D="$(cat input/description.md)" \
  && node_modules/.bin/pva bilibili upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & node_modules/.bin/pva douyin  upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & node_modules/.bin/pva kuaishou upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & node_modules/.bin/pva weixin  upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & wait
```

### Parallel Multi-line (easier to read)

```bash
VIDEO="$(pwd)/input/video.mp4"
COVER="$(pwd)/input/cover.png"
TITLE=$(cat input/title.json | jq -r .title)
TAGS=$(cat input/tags.json | jq -r '.tags | join(",")')
DESC=$(cat input/description.md)

node_modules/.bin/pva bilibili upload --video "$VIDEO" --title "$TITLE" --desc "$DESC" --tags "$TAGS" --cover "$COVER" &
node_modules/.bin/pva douyin  upload --video "$VIDEO" --title "$TITLE" --desc "$DESC" --tags "$TAGS" --cover "$COVER" &
node_modules/.bin/pva kuaishou upload --video "$VIDEO" --title "$TITLE" --desc "$DESC" --tags "$TAGS" --cover "$COVER" &
node_modules/.bin/pva weixin  upload --video "$VIDEO" --title "$TITLE" --desc "$DESC" --tags "$TAGS" --cover "$COVER" &

wait
```

### All headed (default)

```bash
pva douyin upload --video ./video.mp4 --title "Title" &
pva kuaishou upload --video ./video.mp4 --title "Title" &
wait
```

## Important Notes

- **Always use headed mode** (visible browser). Never use `--headless` for uploads.
- Each `pva` process launches its own Playwright browser instance — no conflict between platforms.
- Saved auth state is in `~/.local/share/mise/installs/node/22/lib/node_modules/@panda-video-automation/pva/playwright/.auth/`.
- Login only needs to be done once; subsequent uploads reuse saved sessions.
- `pva --help` for full CLI reference.

## 🚨 Pitfalls & Solutions

### 1. `pva` command not found

If running in a local install (not global), the `pva` binary is at `node_modules/.bin/pva`. Do NOT rely on `pva` being in PATH — always use the full relative path:

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
pva bilibili upload --video input/video.mp4 ...

# ✅ Correct — use absolute path
pva bilibili upload --video /path/to/project/input/video.mp4 ...

# ✅ Best — resolve in a bash variable
VIDEO="$(pwd)/input/video.mp4"
COVER="$(pwd)/input/cover.png"
pva bilibili upload --video "$VIDEO" --cover "$COVER" ...
```

### 3. Description with special characters / multi-line text

The `--desc` argument supports multi-line text, but shell escaping can be tricky with Chinese characters and newlines. Prefer reading from file into a variable:

```bash
DESC=$(cat input/description.md)
pva bilibili upload --desc "$DESC" ...
```

## Behavioral Rules

- **Do NOT check login status unless the user explicitly mentions it.** Skip `--check` / login verification steps entirely by default.
- When the user says "upload to [platforms]", directly open upload commands for all listed platforms in **parallel** (using `&` + `wait`).
- Do not ask "are you logged in?" or attempt to verify credentials upfront. If a platform requires login, the browser will prompt for it naturally.
- For the **input folder**: read `input/title.json` for the title, use `input/*.mp4` for the video path.
