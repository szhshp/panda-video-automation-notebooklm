---
name: notebooklm-prep-upload
description: Prepare an externally generated explainer video for upload: select the approved BGM version, create title/description/tags and the cover, then copy everything into output/<slug>/.
---

# Prepare Upload

The video is rendered by the external HyperFrames project, watermarked, and mixed with BGM before this skill runs. This skill turns those working files into the upload-ready tree and never downloads, trims, or re-encodes video.

## Context

Every notebook gets its own folder, named with an **English kebab-case slug** (see [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) → Step 0).

```
input/
  <slug>/
    research-report.md   ← source of truth for title/description/tags/cover
    video.mp4            ← clean master (external project) — never uploaded directly
    video-watermarked.mp4
    video-bgm-v<N>.mp4   ← BGM mixes; the approved one becomes the upload video
    title.json
    description.md
    tags.json
    cover.png
output/
  <slug>/
    video/
      video.mp4          ← upload copy of the approved BGM version
      cover.png          ← upload-ready cover
    spider/
      title.json         ← {"title": "..."}
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
1. **Slug** — if the run is already in progress, reuse `$NB` from earlier steps
2. **Video topic or title** (optional) — if not provided, derive it from `research-report.md`
3. **Description** (optional) — written to `input/$NB/description.md`. Constraints: first line must be a headline title, at most **3 topics**, each line ≤ **50 characters**, **no emoji anywhere** (plain text only, no symbols or emoticons).
4. **Tags** (optional, comma-separated, **max 3**) — set as `VIDEO_TAGS` env var. Most platforms (e.g. Bilibili) limit tags to 3. Pick the most relevant ones.

Do not ask for a source video path: the video must already exist in `input/$NB/` from the render → watermark → BGM chain.

### Step 2: Generate Engaging Title

Read `input/$NB/research-report.md` and decide the title. **Decide it directly — do NOT ask the user to confirm or choose.**

- **Fixed short title ≤12 Chinese characters** — always the main title, no user confirmation
- **Simple & clear** — plain everyday words, core event named directly, easy to understand at a glance
- **Bilibili max 80 characters** — optional full version, decided directly, no confirmation
- The title should be engaging and attract clicks (e.g., use questions, surprising angles, or clear value propositions)

If the user provided a title directly, use their title.

### Step 3: Create Directories

Steps 4–6 work inside `input/$NB/`; Step 7 copies the results into `output/$NB/`. Create both up front:

```bash
mkdir -p "input/$NB" "output/$NB/video" "output/$NB/spider"
```

### Step 4: Create Metadata Files

Write all three metadata files into `input/$NB/` — Step 7 copies them into `output/$NB/spider/`, so a missing file there means an upload with an empty description or no tags.

**4a. `title.json`** — always the **≤12-character short title**:

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

### Step 5: Select the BGM Version

The upload video is a BGM mix, never the clean master. Pick the highest version unless the user names one:

```bash
BGM_FILE="${VIDEO_BGM:-$(ls -1 "input/$NB"/video-bgm-v*.mp4 2>/dev/null | sort -V | tail -1)}"
[ -n "$BGM_FILE" ] || { echo "MISSING: no input/$NB/video-bgm-v*.mp4 — run the video-bgm step first"; exit 1; }
[ -f "$BGM_FILE" ] || { echo "MISSING: $BGM_FILE"; exit 1; }
echo "Upload video: $BGM_FILE"
```

If several versions exist, show the user the list (`video-bgm-v1.mp4`, `video-bgm-v2.mp4`, …) and make sure the chosen one is the one they approved. Publishing without BGM requires an explicit choice: `VIDEO_BGM="input/$NB/video-watermarked.mp4"`.

### Step 6: Generate Minimalist Academic Cover

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

### Step 7: Copy to Output

This is the only step that fills `output/$NB/`. Verify everything first, then copy:

```bash
for f in "$BGM_FILE" "input/$NB/cover.png" "input/$NB/title.json" \
         "input/$NB/description.md" "input/$NB/tags.json"; do
  [ -f "$f" ] || { echo "MISSING: $f — run Step 4/5/6 first"; exit 1; }
done

cp "$BGM_FILE"                "output/$NB/video/video.mp4"
cp "input/$NB/cover.png"      "output/$NB/video/cover.png"
cp "input/$NB/title.json"     "output/$NB/spider/title.json"
cp "input/$NB/description.md" "output/$NB/spider/description.md"
cp "input/$NB/tags.json"      "output/$NB/spider/tags.json"
```

The check runs first so a missing metadata file aborts the step instead of silently publishing with an empty description.

### Step 8: Report

Tell the user:
- ✅ `output/$NB/video/video.mp4` is ready (copied from `<BGM version>`)
- ✅ `output/$NB/spider/title.json` is ready
- ✅ Remind them they can also set `VIDEO_DESC` and `VIDEO_TAGS` before running the upload
- ✅ Ready to run: `npx pva <platform> upload` (see [pva-publisher](.agents/skills/pva-publisher/SKILL.md))
