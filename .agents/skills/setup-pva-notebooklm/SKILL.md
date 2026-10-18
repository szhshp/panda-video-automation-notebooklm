---
name: setup-pva-notebooklm
description: Overview of the Panda Video Automation pipeline — NotebookLM research and report, external explainer video, watermark, BGM, cover, upload. Run this skill to see the full workflow and prerequisites.
---

# Panda Video Automation — Pipeline Overview

This project automates the delivery half of the pipeline: research a topic in NotebookLM and produce a report, hand the report to an external HyperFrames project that renders the explainer video, then watermark it, add background music, prepare the metadata and cover, and publish.

```
Research + report → Explainer video (external) → Watermark → BGM
→ Meta → Cover → Prepare → Publish
```

## Prerequisites

PVA is installed as a **local project dependency** (declared in `package.json`). Run inside the project root:

```bash
npm install
npx pva version
```

> **Note:** PVA is used from the **local project directory** (`node_modules/.bin/pva` or `npx pva`) — never installed globally. Bare `pva` is not on PATH.

Also required:

- `notebooklm-mcp-cli` for research and the report:
  ```bash
  uv tool install notebooklm-mcp-cli   # or: pip install notebooklm-mcp-cli
  ```
- `ffmpeg` / `ffprobe` for watermarking and BGM mixing:
  ```bash
  brew install ffmpeg        # macOS
  # apt install ffmpeg      # Linux
  ```
- Python 3 + Pillow for covers: `pip install Pillow`
- The external HyperFrames checkout (default `~/Documents/Dev/Github/hyperframes`, override with `HYPERFRAMES_ROOT`). It is a separate project and is not an npm dependency of this repo.

> **Note (optional):** `nlm skill install agents` is **optional** — only needed if you want nlm to emit agent-style skill files. The project's own `.agents/skills/` already covers the pipeline; skip unless explicitly requested.

## Pipeline Steps

Each remote step resolves immediately (background work). Wait times are noted.

### Step 1: Research + Report

Creates a NotebookLM notebook, runs deep web research, imports sources, then generates and downloads the report.

```bash
nlm notebook create "<topic>" -j
nlm research start "<query>" -m deep -n <notebook-id>
# after the user returns:
nlm research import <notebook-id>
nlm report create <notebook-id> --language zh-Hans -y -j
nlm download report <notebook-id> -o "input/<slug>/research-report.md"
```

**Wait ~5 min for research, a few more for the report.** `research-report.md` is the source of truth for everything downstream.

### Step 2: Explainer Video (External)

The HyperFrames project (`videos/<slug>`) consumes `research-report.md` and returns a clean 1080p30 master to `input/<slug>/video.mp4` — no watermark, no BGM, no bumper.

### Step 3: Watermark

```bash
ffmpeg -i "input/$NB/video.mp4" -i input/watermark.png \
  -filter_complex "overlay=W-w:H-h" \
  -codec:a copy "input/$NB/video-watermarked.mp4" -y
```

### Step 4: BGM

`/video-bgm` mixes music at 0.6 volume with a 10-second fade-out, writing `input/<slug>/video-bgm-v<N>.mp4`.

### Step 5–6: Metadata and Cover

`title.json`, `description.md`, `tags.json`, then the minimalist academic cover.

### Step 7: Prepare for Upload

Copies the approved BGM version and the metadata into `output/<slug>/`.

```bash
/skill notebooklm-prep-upload
```

### Step 8: Publish to Platforms

Uploads to Bilibili, Douyin, Kuaishou, Weixin Video, or YouTube.

```bash
/skill pva-publisher
```

## Quick Reference

| Skill | What it does |
|-------|-------------|
| `/notebooklm-pipeline` | **[Entry]** Full pipeline |
| `/pva-check` | Verify all prerequisites |
| `/setup-pva-notebooklm` | This overview |
| `/notebooklm-research` | Research a topic and download the report |
| `/video-bgm` | Mix BGM into the watermarked video |
| `/notebooklm-prep-upload` | Prepare the BGM version for upload |
| `/notebooklm-status` | List notebooks and their artifacts |
| `/pipeline-status` | Report pipeline progress |
| `/pva-publisher` | Cross-platform upload |

> **Note:** Remote steps (research, report generation, external render) launch background work and resolve immediately. The user should wait the indicated time before proceeding. Never park a polling loop.
