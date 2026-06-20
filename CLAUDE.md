# Panda Video Automation - NotebookLM

## Agent Skills

This project uses agent skills located in `.agents/skills/`. Each skill is a directory with a `SKILL.md` file defining its behavior, triggers, and workflow steps.

Available skills:
- `notebooklm-pipeline` — **[ENTRY POINT]** Full pipeline: research → video → download → prepare → publish. Always use this skill first for any end-to-end task.
- `notebooklm-research` — NotebookLM deep research workflow (sub-step of pipeline)
- `notebooklm-video` — Create NotebookLM notebook + generate video (sub-step of pipeline)
- `notebooklm-status` — List all notebooks and their artifacts
- `get-notebooklm-video` — Download video artifact from NotebookLM
- `notebooklm-prep-upload` — Prepare video for upload pipeline
- `minimalist-academic-cover` — Generate minimalist academic-style cover images via Python/Pillow
- `pva-publisher` — Cross-platform video upload
- `pva-check` — Prerequisites check
- `setup-pva-notebooklm` — Legacy pipeline overview

