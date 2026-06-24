---
name: notebooklm-video
description: Create a NotebookLM notebook for a topic and generate a video overview in Simplified Chinese. Delegates installation and login checks to notebooklm-research skill.
---

# NotebookLM Video Workflow

Research a topic using NotebookLM and generate a video overview in Simplified Chinese.

## Workflow Steps

When this skill is invoked, follow these steps in order:

### Step 1: Prerequisites

First, run the [notebooklm-research](.claude/skills/notebooklm-research/SKILL.md) skill to handle installation check, login verification, and setup.

### Step 2: Ask for Research Topic

Prompt the user: "What research topic would you like to explore?"

Wait for the user to provide the research title/topic.

### Step 3: Create Notebook and Set Context

1. Create a new notebook with the research topic as title:
   ```bash
   notebooklm create "<research-title>" --json
   ```
   Parse the notebook ID from the JSON output.

2. Set the notebook as current context:
   ```bash
   notebooklm use <notebook-id>
   ```

### Step 4: Generate Video (No Wait)

1. Ask the user to confirm whether they want to proceed with video generation.
2. If confirmed, generate a video overview in Simplified Chinese (no wait — runs in background):
   ```bash
   notebooklm generate video "<research-title> video overview" --language zh_Hans --format explainer --style auto --no-wait --json
   ```
3. Report the results to the user with the video ID.
4. Tell the user the video is being generated in the background — check back in about 5 minutes.
5. If declined, skip video generation and report notebook creation only.

## Example

User: `/notebooklm-video`

Agent:
1. ✅ Runs notebooklm-research skill for prerequisites
2. ❓ Asks: "What research topic would you like to explore?"
3. User: "How much longer can the AI bubble last?"
4. ✅ Creates notebook and sets context
5. ✅ Generates video overview in Simplified Chinese
6. Reports back with notebook ID and video info
