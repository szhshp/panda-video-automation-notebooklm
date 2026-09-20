---
name: notebooklm-prep-upload
description: Prepare a downloaded NotebookLM video for upload: create the title JSON file and move the video to the correct path expected by the Bilibili/Douyin upload e2e scripts.
---

# NotebookLM Prepare Upload

After downloading a NotebookLM video, prepare it for the upload pipeline by creating the metadata JSON and placing the video file in the expected location.

## Context

Every notebook gets its own folder, named with an **English kebab-case slug** (see [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) → Step 0). The upload scripts expect this structure under the project root:

```
input/
  <slug>/
    video.mp4          ← video file
    title.json         ← {"title": "..."}
    description.md
    tags.json
    cover.png
output/
  <slug>/
    video/
      video.mp4        ← upload-ready video
      cover.png        ← upload-ready cover
    spider/
      title.json       ← {"title": "..."}
      description.md
      tags.json
```

Set the slug once, then reuse it:

```bash
NB=zhipu-zcode
```

## Workflow Steps

### Step 1: Gather Input

Ask the user for:
1. **Source video path** — where the downloaded video is located
2. **Video topic or title** (optional) — if the user doesn't provide one, infer the topic from the filename automatically
3. **Description** (optional) — written to `input/$NB/description.md`. Constraints: first line must be a headline title, at most **3 topics**, each line ≤ **50 characters**, **no emoji anywhere** (plain text only, no symbols or emoticons).
4. **Tags** (optional, comma-separated, **max 3**) — will be set as `VIDEO_TAGS` env var. Most platforms (e.g. Bilibili) limit tags to 3. Pick the most relevant ones.

### Step 2: Generate Engaging Title

If the user didn't provide a title, infer the topic from the filename. Then generate an attractive, clickable title suitable for Bilibili/Douyin. **Decide the title directly — do NOT ask the user to confirm or choose.** Keep these constraints in mind:
- **Fixed short title ≤12 Chinese characters** — always the main title, no user confirmation
- **Simple & clear** — plain everyday words, core event named directly, easy to understand at a glance
- **Bilibili max 80 characters** — optional full version, decided directly, no confirmation
- The title should be engaging and attract clicks (e.g., use questions, surprising angles, or clear value propositions)

If the user provided a title directly, skip generation and use their title directly.

No confirmation step — proceed straight to writing files with the decided short title.

### Step 3: Create Directories

Steps 4–7 work inside `input/$NB/` (the working area); Step 8 then copies the results into `output/$NB/` (upload-ready). Create both up front:

```bash
mkdir -p "input/$NB" "output/$NB/video" "output/$NB/spider"
```

### Step 4: Create Metadata Files

Write all three metadata files into `input/$NB/` — Step 8 copies them into `output/$NB/spider/`, so a missing file there means an upload with an empty description or no tags.

**4a. `title.json`** — always the **≤12-character short title** (the fixed short-title rule; the ≤80-char Bilibili version is an optional extra, never the file's main title):

```json
{
  "title": "<short-title>"
}
```

**4b. `description.md`** — first line = headline title, then ≤3 topic lines, each **≤50 characters**, Simplified Chinese, **no emoji**, plain numbered list (`1.` `2.` `3.` — not ①②③):

```markdown
<short-title>

视频看点：
1. <topic-1>
2. <topic-2>
3. <topic-3>
```

**4c. `tags.json`** — ≤3 most relevant tags:

```json
{
  "tags": ["<tag1>", "<tag2>"]
}
```

### Step 5: Move Video

Copy the source video into the notebook's folder:

```bash
SRC="<source-path>"
DEST="input/$NB/video.mp4"
# Skip when the video is already in place — get-notebooklm-video downloads to
# exactly this path, and `cp` onto itself fails with "are identical".
[ "$SRC" -ef "$DEST" ] || cp "$SRC" "$DEST"
```

> **Don't trim twice.** If `get-notebooklm-video` already ran, the video is **already trimmed** — skip Step 6 and go straight to Step 7. That skill does **not** watermark, so if the watermark is still missing, apply it first (see [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) → Step 3b). Every published video must carry the watermark, and it must be applied **before** trimming.

### Step 6: Trim Last 4 Seconds

NotebookLM-generated videos often include an ending bumper/logo frame. Trim the last 4 seconds from the video before upload using ffmpeg:

```bash
# Get video duration in seconds
DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "input/$NB/video.mp4")

# Calculate new duration (minus 4 seconds)
TRIM_TO=$(python3 -c "print(f'{$DURATION-4:.3f}')")

# Cut video to new duration (fast — stream copy, no re-encode)
ffmpeg -i "input/$NB/video.mp4" -t "$TRIM_TO" -c copy \
  "input/$NB/video-trimmed.mp4"

# Replace original with trimmed version
mv "input/$NB/video-trimmed.mp4" "input/$NB/video.mp4"

echo "Trimmed last 4s: ${DURATION}s → ${TRIM_TO}s"
```

> **Requires `ffmpeg` and `ffprobe`.** Install via `brew install ffmpeg` (macOS) or `apt install ffmpeg` (Linux).

### Step 7: Generate Minimalist Academic Cover

Use the [minimalist-academic-cover](.agents/skills/minimalist-academic-cover/SKILL.md) skill to generate a cover via Python/Pillow instead of extracting a video frame.

The cover skill will:
1. Design a minimalist academic cover based on the video title
2. Generate the cover using Python/Pillow
3. Save the completed cover to `input/$NB/cover.png`

This replaces the old `ffmpeg` frame-extraction with a professional minimalist academic design.

**Design rules** (see full spec in the cover skill):
- Solid **black (#000000)** background
- Large bold centered title (sans-serif, dead center) — bright yellow #FFD700, 130px
- Subtitle below at 86px (same yellow), light gray caption at the bottom
- No logos, dates, author names, or decoration

### Step 8: Copy to Output

Steps 4–7 wrote into `input/$NB/`; the upload scripts read from `output/$NB/`. This step is what fills it:

```bash
for f in video.mp4 cover.png title.json description.md tags.json; do
  [ -f "input/$NB/$f" ] || { echo "MISSING: input/$NB/$f — run Step 4/5/7 first"; exit 1; }
done

cp "input/$NB/video.mp4"      "output/$NB/video/video.mp4"
cp "input/$NB/cover.png"      "output/$NB/video/cover.png"
cp "input/$NB/title.json"     "output/$NB/spider/title.json"
cp "input/$NB/description.md" "output/$NB/spider/description.md"
cp "input/$NB/tags.json"      "output/$NB/spider/tags.json"
```

The check runs first so a missing metadata file aborts the step instead of silently publishing with an empty description.

### Step 9: Report

Tell the user:
- ✅ `output/$NB/video/video.mp4` is ready
- ✅ `output/$NB/spider/title.json` is ready
- ✅ Remind them they can also set `VIDEO_DESC` and `VIDEO_TAGS` before running the upload
- ✅ Ready to run: `npx pva <platform> upload` (see [pva-publisher](.agents/skills/pva-publisher/SKILL.md))
