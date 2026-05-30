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

### Parallel Upload

```bash
TITLE=$(cat /Users/Zill/Documents/Dev/Github/panda-video-automation-notebooklm/input/title.json | jq -r .title)

pva bilibili upload --video /Users/Zill/Documents/Dev/Github/panda-video-automation-notebooklm/input/video.mp4 --title "$TITLE" &
pva douyin  upload --video /Users/Zill/Documents/Dev/Github/panda-video-automation-notebooklm/input/video.mp4 --title "$TITLE" &
pva kuaishou upload --video /Users/Zill/Documents/Dev/Github/panda-video-automation-notebooklm/input/video.mp4 --title "$TITLE" &
pva weixin  upload --video /Users/Zill/Documents/Dev/Github/panda-video-automation-notebooklm/input/video.mp4 --title "$TITLE" &

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

## Behavioral Rules

- **Do NOT check login status unless the user explicitly mentions it.** Skip `--check` / login verification steps entirely by default.
- When the user says "upload to [platforms]", directly open upload commands for all listed platforms in **parallel** (using `&` + `wait`).
- Do not ask "are you logged in?" or attempt to verify credentials upfront. If a platform requires login, the browser will prompt for it naturally.
- For the **input folder**: read `input/title.json` for the title, use `input/*.mp4` for the video path.
