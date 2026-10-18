---
name: notebooklm-research
description: Research a topic with NotebookLM — create notebook, run deep research, auto-import sources, then generate and download a research report to input/<slug>/research-report.md. Checks nlm installation, login, and skill installation first.
---

# NotebookLM Research Workflow

Research a topic using NotebookLM: create a notebook, run deep web research with auto-import sources, then generate and download the report that every later pipeline step depends on.

## Workspace

Every run works inside its own slug folder (see [notebooklm-pipeline](.agents/skills/notebooklm-pipeline/SKILL.md) → Step 0):

```bash
NB=zhipu-zcode
mkdir -p "input/$NB"
```

All paths below assume `$NB` is set. The report lands at `input/$NB/research-report.md`.

## Core Principle: Non-Blocking

**Never** run blocking commands in main conversation. Research and generation take minutes. Instead:

1. **Launch the operation** (non-blocking by default)
2. **Immediately return to the user** — tell them what's running and how long it'll take
3. **The user will check back later** and re-invoke if needed

### Never park a waiting process

The wait belongs to the **user**, not to the agent. After launching a remote operation (research, import, video generation), report the ETA and **end the turn**. Do NOT keep yourself busy waiting for it. Concretely, never:

- Run a background polling loop (`while` / `for` + `sleep`, `until ... done`)
- Arm a monitor / watch task that fires when the remote status changes
- Block on a long-poll flag such as `nlm research status <id> --max-wait 300`
- Tell the user "I'll continue automatically once it finishes" — you won't, and the parked process just burns tokens in the background

Every status check in this project is a **single, instant snapshot**:

```bash
nlm research status <notebook-id> --max-wait 0   # returns immediately, never blocks
nlm studio status <notebook-id> -j               # instant
```

If the operation is still running, say so and let the user come back. One snapshot per user request — not a loop.

## Workflow Steps

When this skill is invoked, follow these steps in order:

### Prerequisites: Check Environment

Before starting, verify prerequisites:

1. **Check NotebookLM Installation**
   ```bash
   nlm --version 2>/dev/null || pip3 show notebooklm-mcp-cli 2>/dev/null
   ```
   If not installed:
   ```bash
   pip install notebooklm-mcp-cli
   nlm skill install agents
   ```

2. **Check Login Status**
   ```bash
   nlm login --check 2>&1
   ```
   If not logged in, tell the user to run `nlm login` first and stop.

3. **Check NotebookLM Skill Installation**
   ```bash
   ls ~/.claude/skills/notebooklm/SKILL.md 2>/dev/null && echo "Skill installed"
   ```
   If not installed:
   ```bash
   nlm skill install agents
   ```

### Step 1: Create Notebook & Start Research

1. If no topic is provided, ask the user: "What research topic would you like to explore?"

2. Create a new notebook with the research topic as title:
   ```bash
   nlm notebook create "<research-title>" -j
   ```
   Parse the notebook ID from the JSON output (`jq -r .notebook.id`).

3. Start deep web research (non-blocking):
   ```bash
   nlm research start "<research-query>" -m deep -n <notebook-id>
   ```

5. **Immediately return to the user with:**
   - Notebook title and ID
   - Research has started in background (deep mode takes ~3-5 minutes)
   - Tell the user to wait and then come back to check progress and import sources
   - The user can later run `nlm research status <notebook-id> --max-wait 0` to check

### Step 2: Check Research Status (User Returns)

When the user comes back to check:

1. Check research status:
   ```bash
   nlm research status <notebook-id> --max-wait 0
   ```

2. If research is still in progress → tell the user it's still running, wait a bit more.

3. If research is complete → show the found sources and ask the user whether to proceed with importing.

### Step 3: Import Sources (User Confirms)

1. **Do NOT run `research import` in main conversation.** Instead:
   - Tell the user this step takes a few minutes
   - Suggest running: `nlm research import <notebook-id>` and waiting for it to complete
   - The user can come back after running it

2. Alternatively, check if sources are already imported:
   ```bash
   nlm source list <notebook-id>
   ```
   If sources are already there and ready → proceed to Step 5.

### Step 4: Wait for Source Processing (User-Initiated)

- Tell the user that after importing, sources need ~1-2 minutes to process
- User can check with: `nlm source list <notebook-id>` (look for "ready" status)
- Come back when all key sources show "ready"

### Step 5: Generate and Download the Research Report

Generate a report from the imported sources, then download it into the slug folder:

```bash
nlm report create <notebook-id> --language zh-Hans -y -j
nlm studio status <notebook-id> -j                # one snapshot — the report appears when completed
nlm download report <notebook-id> -o "input/$NB/research-report.md"
```

The default format is `Briefing Doc`. When the downstream video needs a specific structure (executive summary, numbered themes, data table, implications), use a custom report instead:

```bash
nlm report create <notebook-id> --format "Create Your Own" \
  --prompt "<structure instructions>" --language zh-Hans -y -j
```

Report generation takes a few minutes. Follow the non-blocking rule: snapshot once, report the status, and let the user come back. Never park a polling loop.

`input/$NB/research-report.md` is the **single source of truth** for the rest of the pipeline — the external video project, the title/description/tags, and the cover text all trace back to it. Do not overwrite an existing report without asking the user.

## Example

User: `/notebooklm-research Russian space program 2026`

Agent:
1. ✅ Checks nlm installed (v0.4.1)
2. ✅ Verifies login status (authenticated)
3. ✅ Confirms skill installed
4. ✅ Creates notebook "Russian space program 2026", starts deep research
5. ✅ **Returns immediately** — tells user "Research started (~5 min), come back later to check"
6. 🔄 User returns → agent checks status → shows found sources → asks user to import
7. 🔄 User imports → comes back → agent verifies, generates the report, downloads it to `input/<slug>/research-report.md`
