---
name: notebooklm-research
description: Research a topic with NotebookLM — create notebook, run deep research, auto-import sources, and summarize. Checks notebooklm installation, login, and skill installation first.
---

# NotebookLM Research Workflow

Research a topic using NotebookLM: create a notebook, run deep web research with auto-import sources, and summarize.

## Core Principle: Non-Blocking

**Never** use `notebooklm research wait`, `notebooklm source wait`, or `notebooklm artifact wait` in main conversation. These commands block for minutes. Instead:

1. **Launch the operation** (e.g., `--no-wait` flag)
2. **Immediately return to the user** — tell them what's running and how long it'll take
3. **The user will check back later** and re-invoke if needed

## Workflow Steps

When this skill is invoked, follow these steps in order:

### Prerequisites: Check Environment

Before starting, verify prerequisites:

1. **Check NotebookLM Installation**
   ```bash
   notebooklm --version 2>/dev/null || pip3 show notebooklm-py 2>/dev/null
   ```
   If not installed:
   ```bash
   pip install notebooklm-py
   notebooklm skill install
   ```

2. **Check Login Status**
   ```bash
   notebooklm status 2>&1
   ```
   If authentication fails or session expired:
   ```bash
   notebooklm auth check 2>&1
   ```
   If not logged in, tell the user to run `notebooklm login` first and stop.

3. **Check NotebookLM Skill Installation**
   ```bash
   ls ~/.claude/skills/notebooklm/SKILL.md 2>/dev/null && echo "Skill installed"
   ```
   If not installed:
   ```bash
   notebooklm skill install
   ```

### Step 1: Create Notebook & Start Research

1. If no topic is provided, ask the user: "What research topic would you like to explore?"

2. Create a new notebook with the research topic as title:
   ```bash
   notebooklm create "<research-title>" --json
   ```
   Parse the notebook ID from the JSON output.

3. Set the notebook as current context:
   ```bash
   notebooklm use <notebook-id>
   ```

4. Start deep web research (non-blocking):
   ```bash
   notebooklm source add-research "<research-query>" --mode deep --no-wait
   ```

5. **Immediately return to the user with:**
   - Notebook title and ID
   - Research has started in background (deep mode takes ~3-5 minutes)
   - Tell the user to wait and then come back to check progress and import sources
   - The user can later run `notebooklm research status -n <notebook-id>` to check

### Step 2: Check Research Status (User Returns)

When the user comes back to check:

1. Check research status:
   ```bash
   notebooklm research status -n <notebook-id>
   ```

2. If research is still in progress → tell the user it's still running, wait a bit more.

3. If research is complete → show the found sources and ask the user whether to proceed with importing.

### Step 3: Import Sources (User Confirms)

1. **Do NOT run `research wait --import-all` in main conversation.** Instead:
   - Tell the user this step takes a few minutes
   - Suggest running: `notebooklm research wait --import-all -n <notebook-id>` and waiting for it to complete
   - The user can come back after running it

2. Alternatively, check if sources are already imported:
   ```bash
   notebooklm source list -n <notebook-id>
   ```
   If sources are already there and ready → proceed to Step 5.

### Step 4: Wait for Source Processing (User-Initiated)

- Tell the user that after importing, sources need ~1-2 minutes to process
- User can check with: `notebooklm source list -n <notebook-id>` (look for "ready" status)
- Come back when all key sources show "ready"

### Step 5: Show General Description

Ask NotebookLM to generate a general description/summary of the notebook content:

```bash
notebooklm ask "Summarize the core content and key findings of this notebook in Chinese" -n <notebook-id>
```

Present the generated summary to the user.

## Example

User: `/notebooklm-research Russian space program 2026`

Agent:
1. ✅ Checks notebooklm installed (v0.4.1)
2. ✅ Verifies login status (authenticated)
3. ✅ Confirms skill installed
4. ✅ Creates notebook "Russian space program 2026", starts deep research
5. ✅ **Returns immediately** — tells user "Research started (~5 min), come back later to check"
6. 🔄 User returns → agent checks status → shows found sources → asks user to import
7. 🔄 User imports → comes back → agent verifies and generates summary
