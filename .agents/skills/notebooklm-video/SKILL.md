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

### Step 3: Create Notebook

1. Create a new notebook with the research topic as title:
   ```bash
   nlm notebook create "<research-title>" -j
   ```
   Parse the notebook ID from the JSON output (`jq -r .notebook.id`).

### Step 4: Generate Video

1. Ask the user to confirm whether they want to proceed with video generation.
2. If confirmed, generate a video overview in Simplified Chinese:
   ```bash
   nlm video create <notebook-id> --format explainer --style auto_select --language zh_Hans -y -j
   ```
3. Report the results to the user with the artifact info.
4. Tell the user the video is being generated — check back in about 5 minutes.
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
