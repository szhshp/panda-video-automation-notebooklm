---
name: pipeline-status
description: "Report pipeline progress as a formatted ✅/⚠️ checklist. Trigger: user asks 'status', 'check progress', 'what's running', or 'which pipeline is running'."
---

# Pipeline Status Report

Report progress of all active [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) runs as a formatted checklist.

## Workflow Steps

When this skill is invoked, follow these steps in order:

### Step 1: Query Live State

For each active pipeline, take one instant snapshot (no waiting):

```bash
nlm research status <notebook-id> --max-wait 0   # research snapshot
nlm studio status <notebook-id> -j               # report artifact status
```

`--max-wait 0` is required: it defaults to **300**, so a bare `nlm research status` parks the command for up to 5 minutes and violates the pipeline's "No Background Waiting" rule.

Then inspect the local artifacts, which are the real source of truth for the video stages:

```bash
NB=<slug>
ls -l "input/$NB/research-report.md"     # 1 research report
ls -l "input/$NB/video.mp4"              # 2 clean master from the external project
ls -l "input/$NB/video-watermarked.mp4"  # 3 watermark
ls -1 "input/$NB"/video-bgm-v*.mp4       # 4 BGM versions
ls -l "input/$NB/title.json" "input/$NB/description.md" "input/$NB/tags.json"  # 5
ls -l "input/$NB/cover.png"              # 6
ls -l "output/$NB/video/video.mp4" "output/$NB/video/cover.png" \
      "output/$NB/spider/"*.json "output/$NB/spider/description.md"            # 7
```

Watermark cannot be detected from filenames — if `video-watermarked.mp4` is missing but `video.mp4` looks watermarked, say it is ambiguous instead of guessing.

### Step 2: Report Progress Checklist

Output a formatted list of all pipeline steps. Every pipeline run must report progress in this exact shape:

```text
Notebook #{Index}: {Title}

✅ 1. Research + report (research-report.md)
✅ 2. Explainer video (external render)
✅ 3. Watermark
⚠️ 4. BGM
⚠️ 5. Meta files (title/desc/tags)
⚠️ 6. Cover image
⚠️ 7. Prepare upload
⚠️ 8. Publish
```

Rules:
- One line per pipeline step, numbered in pipeline order (1-8). Step 0 (workspace) is implied by the folder existing and is not listed.
- ✅ = step completed.
- ⚠️ = step pending / in progress (brief note after the line).
- No decorative headers, no raw JSON dumps — just the checklist.
- Include live artifact/notebook IDs on the completed step's line only when useful.
- Multiple pipelines: one block per pipeline, labeled `Notebook #{Index}: {Title}`.

### Step 3: Surface What Needs Attention

After the checklist, one line stating the blocking next action per pipeline (e.g. "P2 next: import sources", "P3 next: run video-bgm"). Ask user which to continue if choices exist.

## Example

User: "check status"

Agent:
1. Takes one research/studio snapshot per active notebook.
2. Checks which local files exist (`research-report.md`, `video.mp4`, `video-watermarked.mp4`, `video-bgm-v*.mp4`, metadata, cover, `output/<slug>/`).
3. Reports:
```text
Notebook #1: 法国骚乱

✅ 1. Research + report (research-report.md)
✅ 2. Explainer video (external render)
✅ 3. Watermark
✅ 4. BGM — video-bgm-v1.mp4
⚠️ 5. Meta files — pending
⚠️ 6. Cover image — pending
⚠️ 7. Prepare upload — pending
⚠️ 8. Publish — pending
```
