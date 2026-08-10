---
name: get-notebooklm-video
description: Download a NotebookLM video artifact to /input/video.mp4 and create a title.json with its title.
---

# Get NotebookLM Video

Download a completed video artifact from a notebook to the local input directory.

> **Note:** Use `nlm download video` (top-level command), NOT `nlm artifact download` (does not exist).

## Workflow Steps

### Step 1: List Artifacts (optional)

```bash
nlm studio status <notebook-id> -j
```

### Step 2: Download Video

Download to `/input/video.mp4`:

```bash
nlm download video <notebook-id> --id <artifact-id> -o input/video.mp4
```

Options:
- `--id <id>` — select by artifact ID
- `-o / --output <path>` — output file path (default: `./{notebook_id}_video.mp4`)
- `--no-progress` — disable progress bar

### Step 3: Create Title Metadata

```bash
echo '{"title": "<video-title>"}' > /input/title.json
```

### Step 4: Trim Last 4 Seconds

NotebookLM-generated videos often include an ending bumper/logo frame. Trim the last 4 seconds from the video to clean it up:

```bash
# Get video duration in seconds
DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "input/video.mp4")

# Calculate new duration (minus 4 seconds)
TRIM_TO=$(echo "$DURATION - 4" | bc)

# Cut video to new duration (fast — stream copy, no re-encode)
ffmpeg -i "input/video.mp4" -t "$TRIM_TO" -c copy \
  "input/video-trimmed.mp4"

# Replace original with trimmed version
mv "input/video-trimmed.mp4" "input/video.mp4"

echo "Trimmed last 4s: ${DURATION}s → ${TRIM_TO}s"
```

> **Requires `ffmpeg` and `ffprobe`.** Install via `brew install ffmpeg` (macOS) or `apt install ffmpeg` (Linux).

## Output

```
input/
├── video.mp4      (typically ~50-100MB, last 4s trimmed)
└── title.json
```
