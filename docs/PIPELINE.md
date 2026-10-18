# Pipeline

End-to-end flow for one video: NotebookLM research → external explainer render → watermark → BGM → metadata → cover → upload.

The agent-facing source of truth is [`.agents/skills/notebooklm-pipeline/SKILL.md`](../.agents/skills/notebooklm-pipeline/SKILL.md). This document is the human-readable map.

## Flow

```
        THIS REPO                              EXTERNAL PROJECT
┌─────────────────────────┐          ┌──────────────────────────────┐
│ 0 Workspace             │          │                              │
│ 1 Research + report ────┼─────────▶│ 2 Explainer video render     │
│                         │          │   (HyperFrames,              │
│                         │◀─────────┼──  faceless-explainer)       │
│ 3 Watermark             │  clean   │                              │
│ 4 BGM                   │  master  └──────────────────────────────┘
│ 5 Meta                  │
│ 6 Cover                 │
│ 7 Prepare upload        │
│ 8 Publish               │
└─────────────────────────┘
```

Everything except step 2 lives in this repository.

## Steps

| # | Step | Owner | Input | Output |
|---|------|-------|-------|--------|
| 0 | Workspace | this repo | topic | `input/<slug>/` |
| 1 | Research + report | this repo (NotebookLM) | topic | `input/<slug>/research-report.md` |
| 2 | Explainer video | external HyperFrames | `research-report.md`, slug | `input/<slug>/video.mp4` (clean master) |
| 3 | Watermark | this repo (ffmpeg) | `video.mp4` | `input/<slug>/video-watermarked.mp4` |
| 4 | BGM | this repo (ffmpeg) | `video-watermarked.mp4`, music | `input/<slug>/video-bgm-v<N>.mp4` |
| 5 | Meta | this repo | `research-report.md` | `title.json`, `description.md`, `tags.json` |
| 6 | Cover | this repo (Pillow) | title + report | `input/<slug>/cover.png` |
| 7 | Prepare upload | this repo | latest `video-bgm-v<N>.mp4` + meta + cover | `output/<slug>/video/*`, `output/<slug>/spider/*` |
| 8 | Publish | this repo (pva) | `output/<slug>/` | platform uploads |

### Step 0 — Workspace

One folder per notebook, English kebab-case slug derived from the topic (`zhipu-zcode`, `ningbo-watermelon`, …). Never use the raw notebook UUID or Chinese text.

### Step 1 — Research + report

```bash
nlm notebook create "<topic>" -j
nlm research start "<query>" -m deep -n <notebook-id>
# user returns:
nlm research import <notebook-id>
nlm report create <notebook-id> --language zh-Hans -y -j
nlm download report <notebook-id> -o "input/<slug>/research-report.md"
```

`research-report.md` is the only source of truth downstream. Every on-screen number, the title, the description, and the cover text trace back to it.

### Step 2 — Explainer video (external)

The external HyperFrames project (`$HYPERFRAMES_ROOT/videos/<slug>`, default `~/Documents/Dev/Github/hyperframes`) consumes the report and renders a 60–120 s, 1920×1080, 30 fps Chinese explainer.

**Return contract — the clean master must be:**

- at `input/<slug>/video.mp4`
- no watermark
- no background music
- no end bumper or logo outro
- narration already mixed

A dirty master is rejected: do not stack a second watermark or mix music over an unknown mix. This repo never calls `nlm video create` and never downloads or trims NotebookLM video.

### Step 3 — Watermark

```bash
ffmpeg -i "input/$NB/video.mp4" -i input/watermark.png \
  -filter_complex "overlay=W-w:H-h" \
  -codec:a copy "input/$NB/video-watermarked.mp4" -y
```

Bottom-right, zero padding, text `熊猫视频自动化引擎` + `https://panda.szhshp.org`. The clean master is never overwritten.

### Step 4 — BGM

```bash
ffmpeg -y -i "input/$NB/video-watermarked.mp4" -stream_loop -1 -i "$BGM" \
  -filter_complex "[1:a]atrim=start=0:end=${DUR},asetpts=PTS-STARTPTS,volume=0.6,afade=t=out:st=${FADE}:d=10[bgm];[0:a][bgm]amix=inputs=2:duration=first:dropout_transition=0:normalize=0,alimiter=limit=0.95:level=disabled[a]" \
  -map 0:v:0 -map "[a]" -c:v copy -c:a aac -b:a 192k -ar 48000 -movflags +faststart \
  "input/$NB/video-bgm-v<N>.mp4"
```

- volume `0.6` (−4.4 dB)
- fade start = `duration − 10`, fade length `10`
- `normalize=0` keeps the narration at its original level
- versioned output; retries increment `N` and never overwrite

### Steps 5–6 — Metadata and cover

- `title.json` — one short title, ≤12 Chinese characters, decided by the agent
- `description.md` — headline line, then `视频看点：` and at most 3 numbered lines, each ≤50 characters, no emoji
- `tags.json` — at most 3 tags
- `cover.png` — 1920×1080, black background, yellow title, gray caption

### Step 7 — Prepare upload

Selects the highest `video-bgm-v<N>.mp4` (or `$VIDEO_BGM`) and fills the upload tree:

```
output/<slug>/video/video.mp4     # copy of the BGM version
output/<slug>/video/cover.png
output/<slug>/spider/title.json
output/<slug>/spider/description.md
output/<slug>/spider/tags.json
```

No BGM file means the step aborts. Nothing is uploaded from `input/`.

### Step 8 — Publish

Parallel upload via the local `pva` dependency. Read each platform's own log — the aggregate exit code is not a verdict.

## Gates

1. **Clean master only** — reject a pre-watermarked, pre-scored, or bumper-ended video at step 2.
2. **Watermark always** — every published video carries it.
3. **BGM always** — 0.6 volume, 10 s fade; skipping requires an explicit user choice.
4. **`output/` only** — uploads read the prepared copies, never `input/`.
5. **Immutable masters** — `video.mp4` and `video-watermarked.mp4` are never modified; BGM versions are append-only.

## Status checklist

```
✅ 1. Research + report
✅ 2. Explainer video (external)
✅ 3. Watermark
⚠️ 4. BGM
⚠️ 5. Meta files
⚠️ 6. Cover image
⚠️ 7. Prepare upload
⚠️ 8. Publish
```

## Legacy folders

Older runs have the watermark baked into `input/<slug>/video.mp4` (applied in place) and no `video-watermarked.mp4`. Treat that file as the watermarked master, skip step 3, and start at step 4.

## What changed from the NotebookLM-only pipeline

| Old | New |
|---|---|
| `nlm video create` | external HyperFrames render |
| `nlm download video` | removed — renderer writes `input/<slug>/video.mp4` |
| trim last 4 s (bumper) | removed — no NotebookLM bumper |
| `nlm describe notebook` summary | `nlm report create` + `download report` |
| watermark in place | `video-watermarked.mp4`, master untouched |
| — | BGM step, versioned mixes |
| prepare copied `input/video.mp4` | prepare copies the approved BGM version |
