---
name: pva-check
description: Check all prerequisites (Python, Node.js, notebooklm-mcp-cli, PVA) before running any workflow.
---

# PVA Prerequisites Check

Verify all required dependencies are installed before running any workflow.

## Workflow Steps

### Step 1: Check Python 3

```bash
python3 --version
```

### Step 2: Check Node.js 20+

```bash
node --version
```

### Step 3: Check notebooklm-mcp-cli

```bash
nlm --version
```

### Step 4: Check PVA CLI

PVA is a **local project dependency** — run from project root:

```bash
npx pva version   # or: node_modules/.bin/pva version
```
