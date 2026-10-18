# Panda Video Automation - NotebookLM

## Agent Skills

This project uses agent skills located in `.agents/skills/`. Each skill is a directory with a `SKILL.md` file defining its behavior, triggers, and workflow steps.

Pipeline map (human-readable, includes the external render handoff): `docs/PIPELINE.md`

Available skills:
- `notebooklm-pipeline` — **[ENTRY POINT]** Full pipeline: research + report → external explainer video → watermark → BGM → meta → cover → prepare → publish. Always use this skill first for any end-to-end task.
- `notebooklm-research` — NotebookLM deep research + report download (sub-step of pipeline)
- `notebooklm-status` — List all notebooks and their artifacts
- `pipeline-status` — Report pipeline progress as a ✅/⚠️ checklist (triggered on "status" / "check progress")
- `video-bgm` — Mix background music into the watermarked video (0.6 volume, 10s fade-out)
- `notebooklm-prep-upload` — Prepare the approved BGM version for upload
- `minimalist-academic-cover` — Generate minimalist academic-style cover images via Python/Pillow
- `pva-publisher` — Cross-platform video upload
- `pva-check` — Prerequisites check
- `setup-pva-notebooklm` — Pipeline overview and prerequisites

