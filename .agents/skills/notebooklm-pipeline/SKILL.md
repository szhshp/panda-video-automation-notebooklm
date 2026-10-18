---
name: notebooklm-pipeline
description: "[ENTRY POINT] Full pipeline — research + NotebookLM report → external explainer video → watermark → BGM → meta → cover → prepare upload → publish. Single entry point for all end-to-end video tasks."
---

# NotebookLM Pipeline — Master Entry Point

This is the **single entry point** for the complete Panda Video Automation pipeline. Every step is handled directly — no redirections.

## Pipeline Overview

```
Workspace → Research + report → Explainer video (external)
→ Watermark → BGM → Meta (title/desc/tags) → Cover
→ Prepare Upload → Publish
```

> **Division of labour:** steps 0–1 and 3–8 run in this repo. Step 2 (the explainer video) is produced by an external HyperFrames project and handed back as a clean master.

---

## Step 0: Workspace (do this first)

Every notebook gets its own folder under `input/`, named with an **English kebab-case slug** derived from the notebook topic. Finished notebooks stay in place — never move them into an archive bucket, and never reuse another notebook's folder.

```
input/
├── watermark.png           # shared asset, git-tracked — stays at input/ root
├── bgm/                    # optional shared music library (gitignored)
└── <slug>/
    ├── research-report.md  # NotebookLM report — source of truth for every later step
    ├── video.mp4           # clean master from the external project — never modified
    ├── video-watermarked.mp4
    ├── video-bgm-v<N>.mp4  # versioned BGM mixes
    ├── title.json
    ├── description.md
    ├── tags.json
    └── cover.png

output/
└── <slug>/
    ├── video/
    │   ├── video.mp4       # upload copy of the approved BGM version
    │   └── cover.png
    └── spider/
        ├── title.json
        ├── description.md
        └── tags.json
```

**Slug rules:** lowercase ASCII, hyphens, no spaces. Derive from the topic:

| Notebook title | Slug |
|---|---|
| 智谱ZCode被曝偷传用户数据 | `zhipu-zcode` |
| 宁波夜市砸瓜事件 | `ningbo-watermelon` |
| 名古屋机场运动员滞留 | `nagoya-airport` |

Never use the raw notebook UUID or Chinese text as a folder name.

**Set the slug once at the start of a run, then reuse it in every command below:**

```bash
NB=zhipu-zcode
mkdir -p "input/$NB"
```

> All commands below assume `$NB` is set. `watermark.png` is the one file that stays at the `input/` root — it is shared by every run.

---

## Step 1: Research + Report

Use the [notebooklm-research](.agents/skills/notebooklm-research/SKILL.md) skill. The short version:

1. Check NotebookLM installation & login
2. Ask user for the research topic
3. Create notebook and start deep web research (`nlm research start "<topic>" -m deep -n <notebook-id>`)
4. Tell user ~5 min, come back later
5. When the user returns: check status, import all sources (`nlm research import <notebook-id> <task-id>`)
6. Generate and download the report:

```bash
nlm report create <notebook-id> --language zh-Hans -y -j
nlm download report <notebook-id> -o "input/$NB/research-report.md"
```

`research-report.md` is the **single source of truth** for the next steps: it is the only input the external video project gets, and every number on screen, the title, the description, and the cover text must trace back to it. Never overwrite an existing report without asking the user first.

> Non-blocking rules are unchanged: one instant status snapshot per user request, never a polling loop. See [notebooklm-research](.agents/skills/notebooklm-research/SKILL.md) → "Never park a waiting process".

---

## Step 2: Explainer Video (External Project)

**This repo no longer generates, downloads, or trims NotebookLM video.** The explainer video is produced by the external HyperFrames project using the `faceless-explainer` workflow.

**External project location:**

```bash
HF_ROOT="${HYPERFRAMES_ROOT:-/Users/Zill/Documents/Dev/Github/hyperframes}"
PROJECT="$HF_ROOT/videos/$NB"
```

**Handoff contract — out:**

- `input/$NB/research-report.md` (absolute path)
- the slug
- target 60–120 seconds, 1920×1080, Simplified Chinese narration

**Handoff contract — back:**

- lands at `input/$NB/video.mp4`
- 1920×1080, 30 fps
- **clean master**: no watermark, no background music, no end bumper/logo outro
- audio already mixed (narration only)

**Gate the return.** Verify spec and reject a dirty master before continuing:

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate \
  -of default=noprint_wrappers=1 "input/$NB/video.mp4"
```

If the video already carries a watermark, music, or a bumper, stop and report it — do not stack a second watermark or mix music over an unknown mix.

> The external project is **not** an npm dependency of this repo and is **not** in this git repository. If `$PROJECT` is missing, report that the external renderer is unavailable; do not fall back to `nlm video create`.

---

## Step 3: Add Watermark

Apply `input/watermark.png` (shared asset at the `input/` root) to the bottom-right corner of the clean master with zero padding (`W-w:H-h` computes flush position from main video and watermark dimensions). Watermark text: `熊猫视频自动化引擎` + `https://panda.szhshp.org`.

```bash
ffmpeg -i "input/$NB/video.mp4" -i input/watermark.png \
  -filter_complex "overlay=W-w:H-h" \
  -codec:a copy "input/$NB/video-watermarked.mp4" -y
```

**Never overwrite the clean master.** The output is `video-watermarked.mp4`; `video.mp4` stays untouched so watermark and BGM retries never re-encode a derived file.

> **Legacy folders:** older runs applied the watermark in place, so their `video.mp4` already carries it and there is no `video-watermarked.mp4`. Do not watermark those again — treat the existing `video.mp4` as the watermarked master and skip to Step 4.

> `ffmpeg` required: `brew install ffmpeg` (macOS) or `apt install ffmpeg` (Linux)

---

## Step 4: Add Background Music

Use the [video-bgm](.agents/skills/video-bgm/SKILL.md) skill.

Summary of the contract:

- input: `input/$NB/video-watermarked.mp4` (fallback to `video.mp4` for legacy folders whose watermark is already baked in)
- output: `input/$NB/video-bgm-v<N>.mp4`
- music volume **0.6** (−4.4 dB)
- fade out over the **final 10 seconds** of the video
- never overwrite `video.mp4` or `video-watermarked.mp4`; each retry gets the next `N`
- never overwrite an existing `video-bgm-v<N>.mp4`

The BGM is applied outside the external video project. Re-rendering the video does not carry the music — rerun this step after any re-render.

---

## Step 5: Generate Meta Files

### 5a. Generate Engaging Title

Generate a title from `input/$NB/research-report.md` — **do NOT ask the user to confirm or choose**. Decide it yourself and use it immediately.

- **Fixed short title rule:** Always use a short title ≤12 Chinese characters as the main title for all platforms. Never present alternatives or ask the user to pick.
- **Simple & clear:** Title must be easy to understand at a glance — plain everyday words, the core event/point named directly, no jargon, no clever ambiguity.
- **Full version** (≤80 chars) — optional for Bilibili; only used if platform needs a longer title, still decided directly, no user confirmation.

The decided short title goes into `title.json` and as the first line of `description.md`.

### 5b. Create title.json

```bash
echo '{"title": "<decided-short-title>"}' > "input/$NB/title.json"
```

### 5c. Create description.md

Format: first line = headline title, then ≤3 topic lines, each ≤50 characters.

```markdown
<video-title>

视频看点：
1. <topic-1, ≤50 chars>
2. <topic-2, ≤50 chars>
3. <topic-3, ≤50 chars>
```

```bash
cat > "input/$NB/description.md" << 'EOF'
<video-title>

视频看点：
1. <topic-1>
2. <topic-2>
3. <topic-3>
EOF
```

**Constraints:**
- First line is the headline title
- All content must be **Simplified Chinese** — avoid English text in the description
- **No emoji anywhere** — no 📊, no symbols, no emoticons in the description text. Plain text only.
- At most **3 topics** (1. 2. 3.)
- Each topic line ≤ **50 characters** — concise punchline, not full sentences
- Use plain numbered list (`1.` `2.` `3.`), NOT circled numbers (① ② ③)
- Total content should be informative and engaging

**⚠️ When passing to PVA upload commands, use actual line breaks (multi-line string), NOT `\n` escape sequences. Shell treats `\n` as literal text, not newlines. Example:**

```bash
VIDEO_DESC="<video-title>

视频看点：
1. <topic-1>
2. <topic-2>
3. <topic-3>" npx pva bilibili upload
```

### 5d. Create tags.json

Pick ≤3 most relevant tags for the topic.

```bash
cat > "input/$NB/tags.json" << 'EOF'
{
  "tags": ["<tag1>", "<tag2>"]
}
EOF
```

**Constraints:** Max 3 tags. Most platforms (e.g. Bilibili) enforce this limit.

---

## Step 6: Generate Cover Image

Use the [minimalist-academic-cover](.agents/skills/minimalist-academic-cover/SKILL.md) skill to generate a 1920×1080 cover with Python/Pillow from the decided title and the research report.

```bash
NB=zhipu-zcode python3 << 'PYEOF'
# see the full script in minimalist-academic-cover
PYEOF
```

Cover saved to `input/$NB/cover.png`.

---

## Step 7: Prepare for Upload

This step is the **gate** — Step 8 uploads from `output/$NB/`, so nothing reaches a platform until it lands here. Run it only after the video is watermarked, the BGM version is approved, and every metadata file plus the cover exists in `input/$NB/`.

**Select the BGM version.** Use the highest `N` unless the user explicitly names a different version:

```bash
BGM_FILE="${VIDEO_BGM:-$(ls -1 "input/$NB"/video-bgm-v*.mp4 2>/dev/null | sort -V | tail -1)}"
[ -n "$BGM_FILE" ] || { echo "MISSING: no input/$NB/video-bgm-v*.mp4 — run the video-bgm step first"; exit 1; }
```

Then copy into the upload tree:

```bash
mkdir -p "output/$NB/video" "output/$NB/spider"
cp "$BGM_FILE"                "output/$NB/video/video.mp4"
cp "input/$NB/cover.png"      "output/$NB/video/cover.png"
cp "input/$NB/title.json"     "output/$NB/spider/title.json"
cp "input/$NB/description.md" "output/$NB/spider/description.md"
cp "input/$NB/tags.json"      "output/$NB/spider/tags.json"
```

> Publishing without BGM is not allowed. If the user explicitly wants a music-free upload, pass `VIDEO_BGM="input/$NB/video-watermarked.mp4"` — the gate still requires an explicit choice.

---

## Step 8: Publish

Upload to desired platform(s) via PVA CLI:

```bash
# One-time login per platform
npx pva weixin login
npx pva bilibili login
npx pva douyin login
npx pva kuaishou login
```

**Check which platforms still have valid auth before uploading:**

```bash
ls -la node_modules/@panda-video-automation/pva/playwright/.auth/
```

> ⚠️ **A missing `<platform>.json` means that platform will fail** — the browser never gets a session and the upload dies with `Error: Not logged in! Please run login script first:`. Auth lives inside `node_modules`, so a dependency reinstall wipes it silently. Report the affected platforms and ask the user to log in; do not run `login` unprompted (it is interactive and deletes the old auth file before saving a new one). Full detail: [pva-publisher](.agents/skills/pva-publisher/SKILL.md) → Pitfalls 4-6. A present-but-stale file can still be rejected — the upload itself is the only real test.

Uploads read from **`output/$NB/`** — the upload-ready copy that Step 7 fills. `input/$NB/` is the working area; never upload from it, or a pre-BGM or unwatermarked intermediate can go out.

Batch upload to all 4 platforms in parallel:

```bash
NB=zhipu-zcode
V="$(pwd)/output/$NB/video/video.mp4"
C="$(pwd)/output/$NB/video/cover.png"
T="$(jq -r .title "output/$NB/spider/title.json")"
G="$(jq -r '.tags | join(",")' "output/$NB/spider/tags.json")"
D="$(cat "output/$NB/spider/description.md")"
LOG="/tmp/pva_upload_${NB}_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$LOG"

# Assign the variables FIRST, then background each upload.
# Do NOT chain assignments into the backgrounded command with `&&` — the
# subshells can start before the variables are set.
for p in bilibili douyin kuaishou weixin; do
  node_modules/.bin/pva "$p" upload \
    --video "$V" --title "$T" --desc "$D" --tags "$G" --cover "$C" \
    > "$LOG/$p.log" 2>&1 &
done
wait

# Always read the per-platform logs — the overall exit code does NOT tell you
# whether each platform succeeded.
for p in bilibili douyin kuaishou weixin; do
  echo "--- $p ---"; tail -8 "$LOG/$p.log"
done
```

> This launches 4 browser instances in parallel, one per platform. Each runs independently without interference.

**Reading the results:** each platform prints its own verdict, and they differ:

| Platform | Success signal | Trust level |
|---|---|---|
| bilibili | `Success: Bilibili` | ✅ asserts success |
| kuaishou | `内容发布成功` | ✅ asserts success |
| weixin | `Success: Weixin` | ✅ asserts success |
| douyin | `Success: Douyin` | ✅ asserts redirect to `/creator-micro/content/manage` (pva ≥ v0.1.12) |
| douyin (old pva) | `Success: Douyin (check manually)` | ⚠️ clicked publish only — verify in the app |

`--video`/`--cover` must be **absolute paths** — the `pva` CLI runs Playwright with its own `cwd`, so relative paths fail.

---

## Output Structure

After pipeline completes:

```
input/
├── watermark.png          (shared watermark source — never moves)
├── bgm/                   (optional shared music library)
└── <slug>/
    ├── research-report.md
    ├── video.mp4          (clean master — untouched)
    ├── video-watermarked.mp4
    ├── video-bgm-v<N>.mp4 (approved BGM mix)
    ├── title.json
    ├── description.md
    ├── tags.json
    └── cover.png

output/
└── <slug>/
    ├── video/
    │   ├── video.mp4      (approved BGM version, ready for upload)
    │   └── cover.png
    └── spider/
        ├── title.json
        ├── description.md
        └── tags.json
```

---

## Usage

**User:** "make a video about [topic]"

**Agent — just do it, no unnecessary questions:**
1. **Step 0** → Workspace: pick the slug, `mkdir -p "input/$NB"`
2. **Step 1** → Research + report (create notebook, deep research, import, `report create` + `download report`)
3. **Step 2** → Hand off to the external HyperFrames project; tell user the ETA; wait for `input/$NB/video.mp4`
4. **Step 3** → Watermark the clean master
5. **Step 4** → BGM (0.6 volume, 10 s fade)
6. **Step 5** → Meta files (title, description, tags)
7. **Step 6** → Cover
8. **Step 7** → Prepare upload (latest BGM version)
9. **Step 8** → Ask user which platform to publish to

**User:** "check progress" / "status"

→ invoke the [pipeline-status](.agents/skills/pipeline-status/SKILL.md) skill.

---

## Status Reporting

When user asks for status/progress, invoke the [pipeline-status](.agents/skills/pipeline-status/SKILL.md) skill and follow its checklist format. The pipeline itself does not own the report format.

## Core Rules

- **One folder per notebook:** every run works inside `input/<slug>/`, named with an English kebab-case slug. Never write pipeline artifacts to the `input/` root, and never overwrite another notebook's folder. `watermark.png` is the single exception — it stays at the `input/` root.
- **Clean master immutable:** `input/<slug>/video.mp4` is written only by the external renderer. Never overwrite, move, trim, or message it in place.
- **BGM versions append-only:** every mix writes the next `video-bgm-v<N>.mp4`; never overwrite an existing version.
- **Non-Blocking:** Launch remote operations and return immediately. Check status later with `nlm research status <id> --max-wait 0` / `nlm studio status <id> -j` — both are instant snapshots.
- **No Background Waiting:** Never park a waiting process. No polling loop, no monitor/watch task, no long-poll flag (`--max-wait 300`), no "I'll continue automatically when it finishes". Launch the operation, tell the user the ETA, then **end the turn**. The user waits and comes back — the agent does not. See [notebooklm-research](.agents/skills/notebooklm-research/SKILL.md) → "Never park a waiting process".
- **No Unnecessary Confirmations:** Proceed directly unless genuinely ambiguous.
- **Watermark Always:** Every video gets the `熊猫视频自动化引擎` + `https://panda.szhshp.org` watermark in the bottom-right corner before BGM and before upload.
- **BGM Always:** Every published video carries the BGM mix (volume 0.6, last 10 s fade-out), unless the user explicitly opts out.
- **Meta Files Always Required:** Every pipeline run must generate `title.json`, `description.md`, `tags.json`, and `cover.png` in `input/<slug>/`.
- **Upload from `output/<slug>/` only:** `input/` holds working files; `output/` holds the upload-ready copies filled by Step 7. Publishing straight from `input/` can ship a pre-BGM or unwatermarked video.
- **No NotebookLM video:** `nlm video create`, `nlm download video`, and the 4-second bumper trim are gone from this pipeline. Do not reintroduce them.
