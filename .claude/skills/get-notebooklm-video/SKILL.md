---
name: get-notebooklm-video
description: Download a NotebookLM video artifact to /input/video.mp4 and create a title.json with its title.
---

# Get NotebookLM Video

Download a completed video artifact from a notebook to the local input directory.

> **Note:** Use `notebooklm download video` (top-level command), NOT `notebooklm artifact download` (does not exist).

## Workflow Steps

### Step 1: List Artifacts (optional)

```bash
notebooklm artifact list -n <notebook-id> --json
```

### Step 2: Download Video

Download to `/input/video.mp4`:

```bash
notebooklm download video -n <notebook-id> -a <artifact-id> input/video.mp4 --force
```

Options:
- `--force` — overwrite existing file
- `--no-clobber` — skip if file exists
- `--all` — download all video artifacts
- `-a / --artifact <id>` — select by artifact ID (supports partial match)
- `--name "<title>"` — select by artifact title (fuzzy match)

### Step 3: Create Title Metadata

```bash
echo '{"title": "<video-title>"}' > /input/title.json
```

## Output

```
input/
├── video.mp4      (typically ~50-100MB)
└── title.json
```
