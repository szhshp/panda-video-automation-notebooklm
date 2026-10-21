---
description: Log in to a platform through the PVA CLI (interactive QR scan in a headed browser)
argument-hint: <bilibili|douyin|kuaishou|weixin|youtube> [more platforms...]
allowed-tools: Bash
---

Log in to one or more publish platforms with the PVA CLI.

## Steps

1. Parse the platforms from `$ARGUMENTS` (space, comma, or newline separated). Normalize aliases:
   - `weixinvideo`, `wechat` → `weixin`
   - `yt` → `youtube`
   - Valid set: `bilibili`, `douyin`, `kuaishou`, `weixin`, `youtube`

   If no platform was given, list the valid platforms and stop — do not guess.

2. Resolve the CLI. Use the project-local binary:

   ```bash
   cd /Users/Zill/Documents/Dev/Github/panda-video-automation-notebooklm && node_modules/.bin/pva --help
   ```

   If `node_modules/.bin/pva` is missing, stop and report that dependencies need installing.

3. Run one platform at a time, in the order given:

   ```bash
   node_modules/.bin/pva <platform> login
   ```

   A headed browser opens and waits for the user to scan the QR code or complete the login. Tell the user which platform is open, then wait — newer versions exit on their own.

   **Warning: never close the browser or interrupt the process mid-login.** The CLI deletes the old auth file before saving the new one, so an aborted login leaves that platform logged out with no cookie file.

4. After each login exits, confirm success from the CLI's own output and from the auth file:

   ```bash
   ls -l node_modules/@panda-video-automation/pva/playwright/.auth/<platform>.json
   ```

   Compare the timestamp with the time the login started. A fresh mtime means the new session was saved.

5. Report per platform: saved / failed / skipped. Do not upload anything in this command.
