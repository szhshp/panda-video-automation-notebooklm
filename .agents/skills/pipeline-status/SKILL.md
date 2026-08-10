---
name: pipeline-status
description: "Report NotebookLM video pipeline progress as a formatted ✅/⚠️ checklist. Trigger: user asks 'status', 'check progress', 'what's running', or 'which pipeline is running'."
---

# Pipeline Status Report

Report progress of all active [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) runs as a formatted checklist.

## Workflow Steps

When this skill is invoked, follow these steps in order:

### Step 1: Query Live State

For each active pipeline, check the current state (no waiting, just a snapshot):

```bash
nlm studio status <notebook-id> -j     # video/artifact status
nlm research status <notebook-id>      # research status
```

Also inspect local artifacts (download/watermark/trim/meta/cover/prepare) to know which post-video steps are done.

### Step 2: Report Progress Checklist

Output a formatted list of all pipeline steps. Every pipeline run must report progress in this exact shape:

```text
Notebook #{Index}: {Title}

✅ 1. Research <notebook title>
✅ 2. Video generation
✅ 3. Download + watermark + trim
⚠️ 4. Meta files (title/desc/tags)
⚠️ 5. Cover image
⚠️ 6. Prepare upload
⚠️ 7. Publish
```

Rules:
- One line per pipeline step, numbered in pipeline order (1-7).
- ✅ = step completed.
- ⚠️ = step pending / in progress (brief note after the line).
- No decorative headers, no raw JSON dumps — just the checklist.
- Include live artifact/notebook IDs on the completed step's line only when useful.
- Multiple pipelines: one block per pipeline, labeled `Notebook #{Index}: {Title}`.

### Step 3: Surface What Needs Attention

After the checklist, one line stating the blocking next action per pipeline (e.g. "P2 next: import sources"). Ask user which to continue if choices exist.

## Example

User: "check status"

Agent:
1. Queries `nlm studio status` for each active notebook.
2. Checks which local files exist (video.mp4, title.json, cover.png, etc.).
3. Reports:
```text
Notebook #1: 同济大学取消教师长期聘任

✅ 1. Research
✅ 2. Video generation
✅ 3. Download + watermark + trim
⚠️ 4. Meta files — pending
⚠️ 5. Cover image — pending
⚠️ 6. Prepare upload — pending
⚠️ 7. Publish — pending
```
