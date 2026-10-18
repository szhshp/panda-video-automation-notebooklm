---
name: video-bgm
description: "Mix background music under a watermarked explainer video at 0.6 volume with a 10-second fade-out, writing a new versioned input/<slug>/video-bgm-v<N>.mp4. Never overwrites the clean master. Trigger: 'BGM', 'background music', '背景音乐', 'add music', '配乐'."
---

# Video BGM

Add background music to an explainer video. The narration stays at its original level, the music sits at 60%, and the track fades out over the final 10 seconds so it lands exactly on the video's last frame.

## Outputs

```
input/<slug>/
├── video.mp4              # clean master — READ ONLY
├── video-watermarked.mp4  # watermark step output — READ ONLY
└── video-bgm-v1.mp4       # this skill
    video-bgm-v2.mp4       # retries append, never overwrite
```

## Workflow Steps

### Step 1: Resolve the slug and the source

```bash
NB=<slug>          # English kebab-case, see notebooklm-pipeline Step 0
SRC="input/$NB/video-watermarked.mp4"
[ -f "$SRC" ] || SRC="input/$NB/video.mp4"
[ -f "$SRC" ] || { echo "MISSING: input/$NB/video.mp4 — run the external render step first"; exit 1; }
```

Prefer the watermarked file. Falling back to `video.mp4` is correct only for a legacy folder where the watermark was applied in place, or when the watermark step was intentionally skipped — never publish an unwatermarked video.

### Step 2: Choose the music track

Require an explicit path; never guess a track:

```bash
BGM="/Volumes/_SHARE/Music/<library>/<track>.mp3"    # or input/bgm/<track>.mp3
[ -f "$BGM" ] || { echo "MISSING: $BGM"; exit 1; }
```

Shared music lives in `input/bgm/` (gitignored). Licensing is the user's responsibility — do not pull tracks from the web without being asked.

### Step 3: Compute the next version

```bash
LAST=$(ls -1 "input/$NB"/video-bgm-v*.mp4 2>/dev/null \
  | sed -E 's/.*video-bgm-v([0-9]+)\.mp4/\1/' | sort -n | tail -1)
N=$(( ${LAST:-0} + 1 ))
OUT="input/$NB/video-bgm-v${N}.mp4"
```

Each run gets a fresh file. This is what makes retries cheap: try a different track, volume, or fade, and compare the versions without losing the previous one.

### Step 4: Measure the video

```bash
DUR=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "$SRC")
FADE=$(python3 -c "print(f'{max(0.0, $DUR - 10):.3f}')")
```

Fade start = duration − 10 s, so the music reaches silence exactly at the last frame.

### Step 5: Mix

```bash
ffmpeg -y -i "$SRC" -stream_loop -1 -i "$BGM" \
  -filter_complex "[1:a]atrim=start=0:end=${DUR},asetpts=PTS-STARTPTS,volume=0.6,afade=t=out:st=${FADE}:d=10[bgm];[0:a][bgm]amix=inputs=2:duration=first:dropout_transition=0:normalize=0,alimiter=limit=0.95:level=disabled[a]" \
  -map 0:v:0 -map "[a]" \
  -c:v copy -c:a aac -b:a 192k -ar 48000 -movflags +faststart \
  "$OUT"
```

Why each piece matters:

- `-stream_loop -1` — loops a track shorter than the video; `atrim` cuts it at the video length.
- `volume=0.6` — the requested 60% (−4.4 dB).
- `afade=t=out:st=<FADE>:d=10` — 10-second fade-out at the end.
- `normalize=0` — **critical**: `amix` defaults to normalizing, which would quietly cut the narration too. Without this flag the 60% music is not 60% and the voice gets quieter.
- `alimiter=limit=0.95:level=disabled` — catches the sum of narration + music before it clips; `level=disabled` keeps the limiter from auto-boosting quiet audio.
- `-c:v copy` — the video stream is copied untouched; only audio is encoded.

### Step 6: Verify

```bash
ffprobe -v error -show_entries format=duration:stream=codec_type,codec_name,duration \
  -of default=noprint_wrappers=1 "$OUT"

ffmpeg -hide_banner -nostats -i "$OUT" -map 0:a -af volumedetect -f null - 2>&1 \
  | grep -E "mean_volume|max_volume"
```

Check:

- duration matches the source (within ~0.05 s)
- video stream is still h264, audio is aac
- peak is below 0 dB (roughly −0.5 dB with the limiter)
- mean volume in the final 10 s is lower than the 10 s before it — the fade is real
- `video.mp4` and `video-watermarked.mp4` mtimes are unchanged

Report the output path to the user. If they want a different track, volume, or fade length, rerun — the next `N` is written automatically.

## Edge Cases

- **No audio stream in the source** — `amix` still works on one input, but check `ffprobe` first; a silent video should get the music as the only track instead of a mix.
- **Track shorter than the video** — covered by `-stream_loop -1`.
- **Video shorter than 10 s** — `FADE` clamps to 0, so the whole track fades from the start.
- **Cover art in the music file** (mp3 with embedded `mjpeg`) — mapping `0:v:0` for the video and only `[a]` for audio avoids dragging the cover image into the output.

## Rules

- Never modify or overwrite `video.mp4` or `video-watermarked.mp4`.
- Never overwrite an existing `video-bgm-v<N>.mp4`.
- Local ffmpeg runs take seconds — no need for background tasks or waiting.
- The BGM is not part of the external HyperFrames project (`audio_meta.json` has a `bgm` slot, unused). A re-render loses the music; rerun this skill after any re-render.
