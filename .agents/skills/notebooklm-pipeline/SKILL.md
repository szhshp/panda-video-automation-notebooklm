---
name: notebooklm-pipeline
description: "[ENTRY POINT] Full NotebookLM pipeline — research → generate video → download → trim → generate meta files → cover → prepare upload → publish. Single entry point for all end-to-end video tasks."
---

# NotebookLM Pipeline — Master Entry Point

This is the **single entry point** for the complete Panda Video Automation pipeline. All work starts here. Every step is handled directly — no redirections.

## Pipeline Overview

```
Research → Video → Download+Trim
→ Meta (title/desc/tags) → Cover
→ Prepare Upload → Publish
```

---

## Step 1: Research

1. Check NotebookLM installation & login
2. Ask user for research topic
3. Create notebook and start deep web research (`notebooklm source add-research "<topic>" --mode deep --no-wait`)
4. Tell user ~5 min, come back later
5. When user returns: check status, import all sources (`notebooklm research wait --import-all -n <id>`)
6. Generate summary via `notebooklm ask "Summarize the core content and key findings of this notebook in Chinese" -n <id>`

## Step 2: Generate Video

Generate video overview in Simplified Chinese (no wait):

```bash
notebooklm generate video "<topic> video overview" --language zh_Hans --format explainer --style auto --no-wait --json -n <notebook-id>
```

Report task ID to user. Tell them ~5 min, come back to check.

**Check status:**

```bash
notebooklm artifact list -n <notebook-id> --json
```

Proceed when status is `completed`.

## Step 3: Download Video + Trim Last 3s

```bash
# 1. Download
notebooklm download video -n <notebook-id> input/video.mp4 --force

# 2. Get video title from artifact list
# Parse artifact title from JSON output

# 3. Create title.json
echo '{"title": "<video-title>"}' > input/title.json

# 4. Trim last 3 seconds (remove NotebookLM bumper)
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
- **Short version** (≤30 chars) — for Douyin
- **Full version** (≤80 chars) — for Bilibili

Present both to user, let them confirm.

### 4b. Create title.json

```bash
echo '{"title": "<confirmed-title>"}' > input/title.json
```

### 4c. Create description.md

Format: first line = headline title, then ≤3 topic lines, each ≤50 characters.

```markdown
<video-title>

📊 Video highlights:
① <topic-1, ≤50 chars>
② <topic-2, ≤50 chars>
③ <topic-3, ≤50 chars>
```

```bash
cat > input/description.md << 'EOF'
<video-title>

📊 Video highlights:
① <topic-1>
② <topic-2>
③ <topic-3>
EOF
```

**Constraints:**
- First line is the headline title
- At most **3 topics** (① ② ③)
- Each topic line ≤ **50 characters** — concise punchline, not full sentences
- Total content should be informative and engaging

**⚠️ When passing to PVA upload commands, use actual line breaks (multi-line string), NOT `\n` escape sequences. Shell treats `\n` as literal text, not newlines. Example:**

```bash
VIDEO_DESC="<video-title>

📊 Video highlights:
① <topic-1>
② <topic-2>
③ <topic-3>" npx pva bilibili upload
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
├── video.mp4          (trimmed last 3s)
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
3. **Step 3** → Download + trim
4. **Step 4** → Generate meta files (titles, description, tags)
5. **Step 5** → Generate cover
6. **Step 6** → Prepare for upload
7. **Step 7** → Ask user which platform to publish to

**User:** "check progress"

**Agent:**
```bash
notebooklm artifact list -n <notebook-id> --json
```

---

## Core Rules

- **Non-Blocking:** All long-running ops use `--no-wait`. Never use `wait` commands in main conversation.
- **No Unnecessary Confirmations:** Proceed directly unless genuinely ambiguous.
- **Meta Files Always Required:** Every pipeline run must generate `title.json`, `description.md`, `tags.json`, and `cover.png` in `input/`.
- **Trim Always:** Every downloaded video gets last 3 seconds trimmed.
