---
name: get-notebooklm-video
description: Download a NotebookLM video artifact into input/<slug>/video.mp4 and create a title.json with its title.
---

# Get NotebookLM Video

Download a completed video artifact from a notebook into that notebook's own folder under `input/`.

> **Note:** Use `nlm download video` (top-level command), NOT `nlm artifact download` (does not exist).

## Workspace (do this first)

Every notebook has its own folder named with an **English kebab-case slug** — never write to the `input/` root, and never reuse another notebook's folder.

```bash
NB=zhipu-zcode
mkdir -p "input/$NB"
```

See [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) → Step 0 for slug rules.

## Workflow Steps

### Step 1: List Artifacts (optional)

```bash
nlm studio status <notebook-id> -j
```

### Step 2: Download Video

```bash
nlm download video <notebook-id> --id <artifact-id> -o "input/$NB/video.mp4"
```

Options:
- `--id <id>` — select by artifact ID
- `-o / --output <path>` — output file path (default: `./{notebook_id}_video.mp4`)
- `--no-progress` — disable progress bar

### Step 3: Create Title Metadata

```bash
echo '{"title": "<video-title>"}' > "input/$NB/title.json"
```

### Step 4: Trim Last 4 Seconds

NotebookLM-generated videos often include an ending bumper/logo frame. Trim the last 4 seconds from the video to clean it up:

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

> **This skill does NOT watermark.** Standalone it produces a trimmed but un-watermarked video. If you are running the full [pipeline](.agents/skills/notebooklm-pipeline/SKILL.md), apply the watermark (its Step 3b) *before* trimming — every published video must carry it.

## Output

```
input/<slug>/
├── video.mp4      (last 4s trimmed — NOT watermarked unless the pipeline ran)
└── title.json
```
