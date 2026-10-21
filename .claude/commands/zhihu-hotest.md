---
description: Fetch the Zhihu hot list (热榜) via the Zhihu CLI and present it as a ranked digest
argument-hint: [limit, default 20]
allowed-tools: Bash
---

Fetch the current Zhihu hot list and present it as a ranked digest.

## Steps

1. Resolve the limit: if `$ARGUMENTS` is a positive integer use it, otherwise use `20`. Ignore any other text.

2. Check CLI readiness first:

   ```bash
   bash /Users/Zill/.claude/skills/zhihu/scripts/run.sh status
   ```

   - If the JSON has `installed: false`, stop and tell the user the CLI is not installed, plus the returned `next_action`. Do not install anything without explicit consent.
   - If `cli.compatible` is `false`, stop and report that a CLI upgrade is required.
   - If `auth.configured` is `false`, stop and report that an Access Secret is needed (`https://developer.zhihu.com/profile`).
   - Otherwise take `cli.binary_path` from the JSON and use that absolute path for the next step. Never use a bare `zhihu-cli` from PATH.

3. Fetch the list:

   ```bash
   "<binary_path>" hot --limit <limit>
   ```

   On `Code != 0`, report the returned `Message` and stop. Do not retry in a loop.

4. Present the results, in the user's language:

   - Numbered list, one entry per item: **bold title** followed by a short one-line gist taken from `Summary` (empty summary means no gist line — never invent one).
   - Put the `Url` link on its own line or inline as the source. Drop the `utm_medium` / `utm_source` query parameters from `Url` for readability.
   - Keep the original `Title` wording; do not rewrite or editorialize headlines.
   - Keep it compact — no grouped sections, no extra commentary unless the user asks for it.

5. If the status JSON had `cli.update_available: true` or `skill.update_available: true`, add one short line at the end noting the available update. Do not upgrade on your own.
