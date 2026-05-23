---
name: pva-check
description: Check all prerequisites (Python, Node.js, notebooklm-py, PVA) before running any workflow.
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

### Step 3: Check notebooklm-py

```bash
notebooklm --version
```

### Step 4: Check PVA CLI

```bash
pva --version
```
