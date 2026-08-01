---
name: notebooklm-research
description: Research a topic with NotebookLM — create notebook, run deep research, auto-import sources, and summarize. Checks nlm installation, login, and skill installation first.
---

# NotebookLM Research Workflow

Research a topic using NotebookLM: create a notebook, run deep web research with auto-import sources, and summarize.

## Core Principle: Non-Blocking

**Never** run blocking commands in main conversation. Research and generation take minutes. Instead:

1. **Launch the operation** (non-blocking by default)
2. **Immediately return to the user** — tell them what's running and how long it'll take
3. **The user will check back later** and re-invoke if needed

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

### Step 5: Show General Description

Ask NotebookLM to generate a general description/summary of the notebook content:

```bash
nlm describe notebook <notebook-id>
```

Present the generated summary to the user.

## Example

User: `/notebooklm-research Russian space program 2026`

Agent:
1. ✅ Checks nlm installed (v0.4.1)
2. ✅ Verifies login status (authenticated)
3. ✅ Confirms skill installed
4. ✅ Creates notebook "Russian space program 2026", starts deep research
5. ✅ **Returns immediately** — tells user "Research started (~5 min), come back later to check"
6. 🔄 User returns → agent checks status → shows found sources → asks user to import
7. 🔄 User imports → comes back → agent verifies and generates summary
