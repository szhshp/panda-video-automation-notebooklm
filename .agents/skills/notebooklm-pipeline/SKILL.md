---
name: notebooklm-pipeline
description: "[ENTRY POINT] Full NotebookLM pipeline — research → generate video → download → watermark → trim → generate meta files → cover → prepare upload → publish. Single entry point for all end-to-end video tasks."
---

# NotebookLM Pipeline — Master Entry Point

This is the **single entry point** for the complete Panda Video Automation pipeline. All work starts here. Every step is handled directly — no redirections.

## Pipeline Overview

```
Workspace → Research → Video → Download → Watermark → Trim
→ Meta (title/desc/tags) → Cover
→ Prepare Upload → Publish
```

---

## Step 0: Workspace (do this first)

Every notebook gets its own folder under `input/`, named with an **English kebab-case slug** derived from the notebook topic. Finished notebooks stay in place — never move them into an archive bucket, and never reuse another notebook's folder.

```
input/
├── watermark.png          # shared asset, git-tracked — stays at input/ root
├── <slug>/
│   ├── video.mp4          # downloaded → watermarked → trimmed
│   ├── title.json
│   ├── description.md
│   ├── tags.json
│   └── cover.png
└── <other-slug>/          # one folder per notebook, side by side

output/
└── <slug>/
    ├── video/
    │   ├── video.mp4
    │   └── cover.png
    └── spider/
        ├── title.json
        ├── description.md
        └── tags.json
```

**Slug rules:** lowercase ASCII, hyphens, no spaces. Derive from the topic:

| Notebook title | Slug |
|---|---|
| 智谱ZCode被曝偷传用户数据 | `zhipu-zcode` |
| 宁波夜市砸瓜事件 | `ningbo-watermelon` |
| 名古屋机场运动员滞留 | `nagoya-airport` |

Never use the raw notebook UUID or Chinese text as a folder name.

**Set the slug once at the start of a run, then reuse it in every command below:**

```bash
NB=zhipu-zcode
mkdir -p "input/$NB"
```

> All commands below assume `$NB` is set. `watermark.png` is the one file that stays at the `input/` root — it is shared by every run.

---

## Step 1: Research

1. Check NotebookLM installation & login
2. Ask user for research topic
3. Create notebook and start deep web research (`nlm research start "<topic>" -m deep -n <notebook-id>`)
4. Tell user ~5 min, come back later
5. When user returns: check status, import all sources (`nlm research import <notebook-id> <task-id>`)
6. Generate summary via `nlm describe notebook <notebook-id>`

## Step 2: Generate Video

Generate video overview in Simplified Chinese:

```bash
nlm video create <notebook-id> --format explainer --style auto_select --language zh_Hans -y -j
```

Report artifact info to user. Tell them ~5 min, come back to check.

**Check status:**

```bash
nlm studio status <notebook-id> -j
```

Proceed when status is `completed`.

## Step 3: Download → Add Watermark → Trim Last 4s

### 3a. Download Video

```bash
nlm download video <notebook-id> -o "input/$NB/video.mp4"
```

### 3b. Add Watermark

Apply `input/watermark.png` (shared asset at the `input/` root) to the bottom-right corner of the video with zero padding (`W-w:H-h` computes flush position from main video and watermark dimensions).

```bash
ffmpeg -i "input/$NB/video.mp4" -i input/watermark.png \
  -filter_complex "overlay=W-w:H-h" \
  -codec:a copy "input/$NB/video-watermarked.mp4" -y
mv "input/$NB/video-watermarked.mp4" "input/$NB/video.mp4"
```

### 3c. Trim Last 4 Seconds (remove NotebookLM bumper)

```bash
DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "input/$NB/video.mp4")
TRIM_TO=$(python3 -c "print(f'{$DURATION-4:.3f}')")
ffmpeg -i "input/$NB/video.mp4" -t "$TRIM_TO" -c copy "input/$NB/video-trimmed.mp4"
mv "input/$NB/video-trimmed.mp4" "input/$NB/video.mp4"
```

> `ffmpeg` required: `brew install ffmpeg` (macOS) or `apt install ffmpeg` (Linux)

## Step 4: Generate Meta Files

### 4a. Generate Engaging Title

Generate a title directly from the artifact title — **do NOT ask the user to confirm or choose**. Decide it yourself and use it immediately.

- **Fixed short title rule:** Always use a short title ≤12 Chinese characters as the main title for all platforms. Never present alternatives or ask the user to pick.
- **Simple & clear:** Title must be easy to understand at a glance — plain everyday words, the core event/point named directly, no jargon, no clever ambiguity.
- **Full version** (≤80 chars) — optional for Bilibili; only used if platform needs a longer title, still decided directly, no user confirmation.

The decided short title goes into `title.json` and as the first line of `description.md`.

### 4b. Create title.json

```bash
echo '{"title": "<decided-short-title>"}' > "input/$NB/title.json"
```

### 4c. Create description.md

Format: first line = headline title, then ≤3 topic lines, each ≤50 characters.

```markdown
<video-title>

视频看点：
1. <topic-1, ≤50 chars>
2. <topic-2, ≤50 chars>
3. <topic-3, ≤50 chars>
```

```bash
cat > "input/$NB/description.md" << 'EOF'
<video-title>

视频看点：
1. <topic-1>
2. <topic-2>
3. <topic-3>
EOF
```

**Constraints:**
- First line is the headline title
- All content must be **Simplified Chinese** — avoid English text in the description
- **No emoji anywhere** — no 📊, no symbols, no emoticons in the description text. Plain text only.
- At most **3 topics** (1. 2. 3.)
- Each topic line ≤ **50 characters** — concise punchline, not full sentences
- Use plain numbered list (`1.` `2.` `3.`), NOT circled numbers (① ② ③)
- Total content should be informative and engaging

**⚠️ When passing to PVA upload commands, use actual line breaks (multi-line string), NOT `\n` escape sequences. Shell treats `\n` as literal text, not newlines. Example:**

```bash
VIDEO_DESC="<video-title>

视频看点：
1. <topic-1>
2. <topic-2>
3. <topic-3>" npx pva bilibili upload
```

### 4d. Create tags.json

Pick ≤3 most relevant tags for the topic.

```bash
cat > "input/$NB/tags.json" << 'EOF'
{
  "tags": ["<tag1>", "<tag2>"]
}
EOF
```

**Constraints:** Max 3 tags. Most platforms (e.g. Bilibili) enforce this limit.

## Step 5: Generate Cover Image

Use Python/Pillow to generate a minimalist academic cover:

```bash
NB=zhipu-zcode python3 << 'PYEOF'
import os
from PIL import Image, ImageDraw, ImageFont

NB = os.environ["NB"]
W, H = 1920, 1080
img = Image.new('RGB', (W, H), color=(0, 0, 0))
draw = ImageDraw.Draw(img)

font_path = '/System/Library/Fonts/STHeiti Light.ttc'  # macOS
# Linux: '/usr/share/fonts/truetype/wqy/wqy-microhei.ttc'

title = '<title-line-1>'
subtitle = '<title-line-2>'
caption = '<caption>'

font_title = ImageFont.truetype(font_path, 130)
font_sub = ImageFont.truetype(font_path, 86)

BRIGHT_YELLOW = (255, 215, 0)
LIGHT_GRAY = (200, 200, 200)
STROKE_COLOR = (30, 20, 0)

# Compute positions and draw...
# (See full script in minimalist-academic-cover skill)

img.save(f'input/{NB}/cover.png')
PYEOF
```

Cover saved to:
- `input/$NB/cover.png`

## Step 6: Prepare for Upload

This step is the **gate** — Step 7 uploads from `output/$NB/`, so nothing reaches a platform until it lands here. Run it only after the video is watermarked, trimmed, and every metadata file plus the cover exists in `input/$NB/`.

```bash
mkdir -p "output/$NB/video" "output/$NB/spider"
cp "input/$NB/video.mp4"      "output/$NB/video/video.mp4"
cp "input/$NB/cover.png"      "output/$NB/video/cover.png"
cp "input/$NB/title.json"     "output/$NB/spider/title.json"
cp "input/$NB/description.md" "output/$NB/spider/description.md"
cp "input/$NB/tags.json"      "output/$NB/spider/tags.json"
```

## Step 7: Publish

Upload to desired platform(s) via PVA CLI:

```bash
# One-time login per platform
npx pva weixin login
npx pva bilibili login
npx pva douyin login
npx pva kuaishou login
```

**Check which platforms still have valid auth before uploading:**

```bash
ls -la node_modules/@panda-video-automation/pva/playwright/.auth/
```

Uploads read from **`output/$NB/`** — the upload-ready copy that Step 6 (or prep-upload's Step 8) fills. `input/$NB/` is the working area; never upload from it, or a half-trimmed or unwatermarked intermediate can go out.

Batch upload to all 4 platforms in parallel:

```bash
NB=zhipu-zcode
V="$(pwd)/output/$NB/video/video.mp4"
C="$(pwd)/output/$NB/video/cover.png"
T="$(jq -r .title "output/$NB/spider/title.json")"
G="$(jq -r '.tags | join(",")' "output/$NB/spider/tags.json")"
D="$(cat "output/$NB/spider/description.md")"
LOG="/tmp/pva_upload_${NB}_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$LOG"

# Assign the variables FIRST, then background each upload.
# Do NOT chain assignments into the backgrounded command with `&&` — the
# subshells can start before the variables are set.
for p in bilibili douyin kuaishou weixin; do
  node_modules/.bin/pva "$p" upload \
    --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
    > "$LOG/$p.log" 2>&1 &
done
wait

# Always read the per-platform logs — the overall exit code does NOT tell you
# whether each platform succeeded.
for p in bilibili douyin kuaishou weixin; do
  echo "--- $p ---"; tail -8 "$LOG/$p.log"
done
```

> This launches 4 browser instances in parallel, one per platform. Each runs independently without interference.

**Reading the results:** each platform prints its own verdict, and they differ:

| Platform | Success signal | Trust level |
|---|---|---|
| bilibili | `Success: Bilibili` | ✅ asserts success |
| kuaishou | `内容发布成功` | ✅ asserts success |
| weixin | `Success: Weixin` | ✅ asserts success |
| douyin | `Success: Douyin (check manually)` | ⚠️ **clicked publish, did not assert** — verify in the app |

`--video`/`--cover` must be **absolute paths** — the `pva` CLI runs Playwright with its own `cwd`, so relative paths fail.

---

## Output Structure

After pipeline completes:

```
input/
├── watermark.png      (shared watermark source — never moves)
└── <slug>/
    ├── video.mp4      (trimmed + watermark applied)
    ├── title.json
    ├── description.md (≤3 topics)
    ├── tags.json      (≤3 tags)
    └── cover.png      (minimalist cover)

output/
└── <slug>/
    ├── video/
    │   ├── video.mp4  (ready for upload)
    │   └── cover.png  (ready for upload)
    └── spider/
        ├── title.json
        ├── description.md
        └── tags.json
```

---

## Usage

**User:** "make a video about [topic]"

**Agent — just do it, no unnecessary questions:**
1. **Step 0** → Workspace: pick the slug, `mkdir -p "input/$NB"`
2. **Step 1** → Research (create, deep research, import, summarize)
3. **Step 2** → Generate video (background, tell user ~5 min)
4. **Step 3** → Download video → Add watermark → Trim last 4s
5. **Step 4** → Generate meta files (titles, description, tags)
6. **Step 5** → Generate cover
7. **Step 6** → Prepare for upload
8. **Step 7** → Ask user which platform to publish to

**User:** "check progress"

**Agent:**
```bash
nlm studio status <notebook-id> -j
```

---

## Status Reporting

When user asks for status/progress, invoke the [pipeline-status](.agents/skills/pipeline-status/SKILL.md) skill and follow its checklist format. The pipeline itself does not own the report format.

## Core Rules

- **One folder per notebook:** every run works inside `input/<slug>/`, named with an English kebab-case slug. Never write pipeline artifacts to the `input/` root, and never overwrite another notebook's folder. `watermark.png` is the single exception — it stays at the `input/` root.
- **Non-Blocking:** Launch operations and return immediately. Check status later with `nlm research status <id> --max-wait 0` / `nlm studio status <id> -j` — both are instant snapshots.
- **No Background Waiting:** Never park a waiting process. No polling loop, no monitor/watch task, no long-poll flag (`--max-wait 300`), no "I'll continue automatically when it finishes". Launch the operation, tell the user the ETA (~5 min), then **end the turn**. The user waits and comes back — the agent does not. See [notebooklm-research](.agents/skills/notebooklm-research/SKILL.md) → "Never park a waiting process".
- **No Unnecessary Confirmations:** Proceed directly unless genuinely ambiguous.
- **Watermark Always:** Every downloaded video gets `熊猫视频自动化引擎` + `https://panda.szhshp.org` watermark in the bottom-right corner before trimming.
- **Meta Files Always Required:** Every pipeline run must generate `title.json`, `description.md`, `tags.json`, and `cover.png` in `input/<slug>/`.
- **Upload from `output/<slug>/` only:** `input/` holds working files; `output/` holds the upload-ready copies filled by Step 6. Publishing straight from `input/` can ship an untrimmed or unwatermarked video.
- **Trim Always:** Every downloaded video gets last 4 seconds trimmed.
