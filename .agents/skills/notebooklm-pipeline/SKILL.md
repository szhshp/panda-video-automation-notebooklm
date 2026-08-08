---
name: notebooklm-pipeline
description: "[ENTRY POINT] Full NotebookLM pipeline — research → generate video → download → watermark → trim → generate meta files → cover → prepare upload → publish. Single entry point for all end-to-end video tasks."
---

# NotebookLM Pipeline — Master Entry Point

This is the **single entry point** for the complete Panda Video Automation pipeline. All work starts here. Every step is handled directly — no redirections.

## Pipeline Overview

```
Research → Video → Download → Watermark → Trim
→ Meta (title/desc/tags) → Cover
→ Prepare Upload → Publish
```

---

## Step 1: Research

1. Check NotebookLM installation & login
2. Ask user for research topic
3. Create notebook and start deep web research (`nlm research start "<topic>" -m deep -n <notebook-id>`)
4. Tell user ~5 min, come back later
5. When user returns: check status, import all sources (`nlm research import <id>`)
6. Generate summary via `nlm describe notebook <id>`

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

## Step 3: Download → Add Watermark → Trim Last 3s

### 3a. Download Video

```bash
nlm download video <notebook-id> -o input/video.mp4
```

### 3b. Add Watermark

Apply `input/watermark.png` to the bottom-right corner of the video (1280×720).

```bash
ffmpeg -i input/video.mp4 -i input/watermark.png \
  -filter_complex "overlay=1055:645" \
  -codec:a copy input/video-watermarked.mp4 -y
mv input/video-watermarked.mp4 input/video.mp4
```

### 3c. Trim Last 3 Seconds (remove NotebookLM bumper)

```bash
DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "input/video.mp4")
TRIM_TO=$(echo "$DURATION - 3" | bc)
ffmpeg -i "input/video.mp4" -t "$TRIM_TO" -c copy "input/video-trimmed.mp4"
mv "input/video-trimmed.mp4" "input/video.mp4"
```

> `ffmpeg` required: `brew install ffmpeg` (macOS) or `apt install ffmpeg` (Linux)

## Step 4: Generate Meta Files

### 4a. Generate Engaging Title

Create two title versions from the artifact title:
- **Short version** (≤12 Chinese chars) — default/main title for all platforms
- **Full version** (≤80 chars) — optional for Bilibili

> **Short title rule:** Always default to a punchy short title ≤12 Chinese characters. Use it as the main title in `title.json` and as the first line of `description.md`.

Present both to user, let them confirm.

### 4b. Create title.json

```bash
echo '{"title": "<confirmed-title>"}' > input/title.json
```

### 4c. Create description.md

Format: first line = headline title, then ≤3 topic lines, each ≤50 characters.

```markdown
<video-title>

📊 视频看点：
1. <topic-1, ≤50 chars>
2. <topic-2, ≤50 chars>
3. <topic-3, ≤50 chars>
```

```bash
cat > input/description.md << 'EOF'
<video-title>

📊 视频看点：
1. <topic-1>
2. <topic-2>
3. <topic-3>
EOF
```

**Constraints:**
- First line is the headline title
- All content must be **Simplified Chinese** — avoid English text in the description
- At most **3 topics** (1. 2. 3.)
- Each topic line ≤ **50 characters** — concise punchline, not full sentences
- Use plain numbered list (`1.` `2.` `3.`), NOT circled numbers (① ② ③)
- Total content should be informative and engaging

**⚠️ When passing to PVA upload commands, use actual line breaks (multi-line string), NOT `\n` escape sequences. Shell treats `\n` as literal text, not newlines. Example:**

```bash
VIDEO_DESC="<video-title>

📊 视频看点：
1. <topic-1>
2. <topic-2>
3. <topic-3>" npx pva bilibili upload
```

### 4d. Create tags.json

Pick ≤3 most relevant tags for the topic.

```bash
cat > input/tags.json << 'EOF'
{
  "tags": ["<tag1>", "<tag2>"]
}
EOF
```

**Constraints:** Max 3 tags. Most platforms (e.g. Bilibili) enforce this limit.

## Step 5: Generate Cover Image

Use Python/Pillow to generate a minimalist academic cover:

```bash
python3 << 'PYEOF'
from PIL import Image, ImageDraw, ImageFont

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

img.save('input/cover.png')
PYEOF
```

Cover saved to:
- `input/cover.png`

## Step 6: Prepare for Upload

```bash
mkdir -p output/video output/spider
cp input/video.mp4 output/video/video.mp4
cp input/title.json output/spider/title.json
cp input/cover.png output/video/cover.png
```

## Step 7: Publish

Upload to desired platform(s) via PVA CLI:

```bash
# One-time login per platform
npx pva weixin login
npx pva bilibili login
npx pva douyin login
npx pva kuaishou login

# Batch upload to all 4 platforms in parallel (single command)
V="$(pwd)/input/video.mp4" && C="$(pwd)/input/cover.png" \
  && T="$(cat input/title.json | jq -r .title)" \
  && G="$(cat input/tags.json | jq -r '.tags | join(",")')" \
  && D="$(cat input/description.md)" \
  && node_modules/.bin/pva bilibili upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & node_modules/.bin/pva douyin upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & node_modules/.bin/pva kuaishou upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & node_modules/.bin/pva weixin upload --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
  & wait
```

> This launches 4 browser instances in parallel, one per platform. Each runs independently without interference.

---

## Output Structure

After pipeline completes:

```
input/
├── video.mp4          (trimmed + watermark applied)
├── watermark.png      (watermark source image)
├── title.json         (title metadata)
├── description.md     (≤3 topics)
├── tags.json          (≤3 tags)
└── cover.png          (minimalist cover)

output/
├── video/
│   ├── video.mp4      (ready for upload)
│   └── cover.png      (ready for upload)
└── spider/
    └── title.json     (ready for upload)
```

---

## Usage

**User:** "make a video about [topic]"

**Agent — just do it, no unnecessary questions:**
1. **Step 1** → Research (create, deep research, import, summarize)
2. **Step 2** → Generate video (background, tell user ~5 min)
3. **Step 3** → Download video → Add watermark → Trim last 3s
4. **Step 4** → Generate meta files (titles, description, tags)
5. **Step 5** → Generate cover
6. **Step 6** → Prepare for upload
7. **Step 7** → Ask user which platform to publish to

**User:** "check progress"

**Agent:**
```bash
nlm studio status <notebook-id> -j
```

---

## Core Rules

- **Non-Blocking:** Launch operations and return immediately. Check status later with `nlm research status` / `nlm studio status`.
- **No Unnecessary Confirmations:** Proceed directly unless genuinely ambiguous.
- **Watermark Always:** Every downloaded video gets `熊猫视频自动化引擎` + `https://panda.szhshp.org` watermark in the bottom-right corner before trimming.
- **Meta Files Always Required:** Every pipeline run must generate `title.json`, `description.md`, `tags.json`, and `cover.png` in `input/`.
- **Trim Always:** Every downloaded video gets last 3 seconds trimmed.
