---
name: setup-pva-notebooklm
description: Overview of the Panda Video Automation pipeline — from NotebookLM research to video upload. Run this skill to see the full workflow.
---

# Panda Video Automation — Pipeline Overview

This project automates the full pipeline: research a topic via NotebookLM → generate AI video → prepare for upload → publish to platforms.

## Prerequisite: Setup

```bash
npm install -g @panda-video-automation/pva
pva --version
```

Also ensure `notebooklm-mcp-cli` is installed:
```bash
pip install notebooklm-mcp-cli
nlm skill install agents
```

## Pipeline Steps

Each step below resolves immediately (background work). Wait times are noted.

### Step 1: Research a Topic

Creates a NotebookLM notebook and starts deep web research.

```bash
nlm notebook create "<topic>" -j
nlm research start "<query>" -m deep -n <notebook-id>
```

**Wait ~5 min** — run `nlm research import <notebook-id>` to finish and auto-import all sources.

### Step 2: Generate Video

Generates a video overview in Simplified Chinese from the notebook.

```bash
nlm video create <notebook-id> --format explainer --style auto_select --language zh_Hans -y -j
```

**Wait ~5 min** — check back and download the video via `/get-notebooklm-video`.

### Step 3: Download Video Artifact

```bash
/skill get-notebooklm-video
```

Downloads video to `/input/video.mp4` with `title.json`.

### Step 4: Prepare for Upload

Moves video and title into the structure expected by upload scripts.

```bash
/skill notebooklm-prep-upload
```

### Step 5: Publish to Platforms

Uploads to Bilibili, Douyin, Kuaishou, Weixin Video, or YouTube.

```bash
/skill pva-publisher
```

## Quick Reference

| Skill | What it does |
|-------|-------------|
| `/pva-check` | Verify all prerequisites |
| `/setup-pva-notebooklm` | This overview |
| `/notebooklm-research` | Research a topic via NotebookLM |
| `/notebooklm-video` | Research + generate video |
| `/get-notebooklm-video` | Download video artifact |
| `/notebooklm-prep-upload` | Prepare video for upload |
| `/pva-publisher` | Cross-platform upload |

> **Note:** Each step launches background work and resolves immediately. The user should wait the indicated time before proceeding to the next step.
