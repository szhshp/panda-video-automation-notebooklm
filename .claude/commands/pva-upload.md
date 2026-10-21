---
description: Upload a prepared pipeline video to one or more platforms; with no platform given, upload to all four
argument-hint: [bilibili|douyin|kuaishou|weixin ...] [slug]
allowed-tools: Bash
---

Upload a prepared video from `output/<slug>/` with the PVA CLI.

## Steps

1. Parse `$ARGUMENTS`. Every token that is a platform name or alias is a platform:
   - `weixinvideo`, `wechat` → `weixin`
   - `yt` → `youtube`
   - Supported here: `bilibili`, `douyin`, `kuaishou`, `weixin`

   Any other token is the notebook **slug**. If no platform is given, the target set is **all four**: `bilibili douyin kuaishou weixin`.

2. Resolve the slug:
   - If a slug was given, use it.
   - Otherwise pick the `output/*/` directory whose `video/video.mp4` was modified most recently, and state which slug was chosen.

   Verify the upload-ready set exists **before** uploading:

   ```bash
   ls output/<slug>/video/video.mp4 output/<slug>/video/cover.png \
      output/<slug>/spider/title.json output/<slug>/spider/description.md \
      output/<slug>/spider/tags.json
   ```

   If any file is missing, stop and report which step of the pipeline is incomplete. Never upload from `input/<slug>/`.

3. Check auth per target platform:

   ```bash
   ls -l node_modules/@panda-video-automation/pva/playwright/.auth/<platform>.json
   ```

   A missing auth file means that platform is skipped with a note to run `/pva-login <platform>`. Do not log in automatically and do not interrupt a login that is in progress.

4. Upload from the project root. Absolute paths are required, and `description.md` must be passed with real newlines:

   ```bash
   cd /Users/Zill/Documents/Dev/Github/panda-video-automation-notebooklm
   NB=<slug>
   V="$(pwd)/output/$NB/video/video.mp4"
   C="$(pwd)/output/$NB/video/cover.png"
   T="$(jq -r .title "output/$NB/spider/title.json")"
   G="$(jq -r '.tags | join(",")' "output/$NB/spider/tags.json")"
   D="$(cat "output/$NB/spider/description.md")"
   LOG="/tmp/pva_upload_${NB}_$(date +%Y%m%d_%H%M%S)"
   mkdir -p "$LOG"

   for p in $(echo "<platforms>"); do
     node_modules/.bin/pva "$p" upload \
       --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
       > "$LOG/$p.log" 2>&1 &
   done
   wait

   for p in $(echo "<platforms>"); do
     echo "--- $p ---"; tail -8 "$LOG/$p.log"
   done
   ```

   Note: the session shell is **zsh**, which does not word-split unquoted variables. Keep the `$(echo "<platforms>")` form (or a literal list) — `for p in $PLATFORMS` sends the whole string as one platform name.

5. Report per platform using its own log — the overall exit code says nothing about individual results:

   | Platform | Success signal | Trust |
   |---|---|---|
   | bilibili | `Success: Bilibili` | asserts success |
   | kuaishou | `内容发布成功` | asserts success |
   | weixin | `Success: Weixin` | asserts success |
   | douyin | `Success: Douyin (check manually)` | clicked publish, did not assert — tell the user to verify in the app |

   Report saved / failed / skipped per platform. Publishing is public, so if the slug was inferred rather than given, confirm it with the user before running the upload.
