---
name: minimalist-academic-cover
description: Generate minimalist academic-style cover images via Python/Pillow — black (#000000) solid background, centered bold white title (120px), thin subtitle (76px) below. ONLY text on black — no graphics, icons, dates, or logos. TRIGGER: activates when user mentions "simple cover", "minimalist cover", "academic cover", or "cover" without further qualifiers. For "detailed cover", "complex cover", "report-style cover", or "detailed cover"/"complex cover", do NOT use this skill — instead generate a detailed infographic with --style professional or --style scientific.
---

# Minimalist Academic Cover Generator

Generate minimalist academic-style video covers using Python/Pillow — pure black (#000000) background with centered white text. No graphics, no NotebookLM infographic generation needed.

## When to Activate vs. When to Use Detailed Infographic

| Trigger | Action | Style |
|---------|--------|-------|
| "simple cover", "minimalist cover", "academic cover" | ✅ Use this skill | `--style editorial` — solid bg, pure text |
| "detailed cover", "complex cover", "report cover", "infographic cover" | ❌ Skip this skill; generate detailed infographic instead | `--style professional` or `--style scientific` — rich graphics, data viz, layout |

## Workflow Steps

### Step 1: Prerequisites

Requires Python 3 with Pillow:
```bash
pip install Pillow
```

Also requires a Chinese-capable font installed. On macOS these are built-in (PingFang, STHeiti). On Linux, install `fonts-wqy-microhei`.

### Step 2: Generate Cover with Python/Pillow

Run the following Python script to generate a 1920×1080 pure black cover with centered white text. Customize `title` and `caption` for the topic.

```python
from PIL import Image, ImageDraw, ImageFont

W, H = 1920, 1080
img = Image.new('RGB', (W, H), color=(0, 0, 0))
draw = ImageDraw.Draw(img)

# Chinese-capable font — adjust path per OS
font_path = '/System/Library/Fonts/STHeiti Light.ttc'  # macOS

title = '[video-title-line-1]'
subtitle = '[video-title-line-2]'
caption = '[subtitle]'

font_title = ImageFont.truetype(font_path, 120)
font_sub = ImageFont.truetype(font_path, 76)

# Title line 1 — centered
bbox = draw.textbbox((0, 0), title, font=font_title)
tw, th = bbox[2] - bbox[0], bbox[3] - bbox[1]
tx = (W - tw) // 2
ty = H // 2 - th - 80
draw.text((tx, ty), title, fill=(255, 255, 255), font=font_title)

# Title line 2 — below with gap
bbox2 = draw.textbbox((0, 0), subtitle, font=font_title)
tw2 = bbox2[2] - bbox2[0]
tx2 = (W - tw2) // 2
ty2 = ty + th + 100
draw.text((tx2, ty2), subtitle, fill=(255, 255, 255), font=font_title)

# Caption — further below
bbox3 = draw.textbbox((0, 0), caption, font=font_sub)
tw3 = bbox3[2] - bbox3[0]
tx3 = (W - tw3) // 2
ty3 = ty2 + 200
draw.text((tx3, ty3), caption, fill=(200, 200, 200), font=font_sub)

img.save('input/cover.png')
```

**Adjust font sizes** by changing the second argument to `ImageFont.truetype()` (title default 120, subtitle default 76). **Adjust spacing** by changing the vertical offsets (`-80`, `+100`, `+200`).

### Step 3: Copy to Standard Path

```bash
cp input/cover.png docs/assets/cover.png
```

### Step 4: Verify Output

Confirm the output file exists at `input/cover.png` and is a valid PNG (check it visually).

---

## Design Specification

### 1. Core Visual Elements

- **Background** — MUST be **solid black (#000000)**. No gradients, textures, decorative lines, or background images. This ensures absolute visual focus.
- **Main Title** — Extra Large font size, **Bold** weight, Sans-serif. Dead-center geometric alignment. White (#FFFFFF).
- **Subtitle** — Font size = **1/4 to 1/5** of main title. **Thin/Light** font weight. Placed below main title. White or very light gray.
- **Strictly forbidden** — graphics, icons, illustrations, decorative lines, dates, author names, logos, watermarks. Only title + subtitle exist on the canvas.

### 2. Generation Logic

- **Semantic Alignment** — The system must receive the core academic/research topic from the notebook's research sources and render the identified thesis as the title.
- **Minimalist Constraint** — Maximize informational negative space. Strip all small text, dates, author names, and logos. Only the title and subtitle are information carriers, delivering maximum visual impact.
- **Defensive Design** — If complex graphic instructions are detected (gradients, images, decorations), fall back to the pure solid-color + text template, ensuring the cover maintains a rigorous academic tone.

### 3. Default Parameters

| Parameter | Value |
|-----------|-------|
| Canvas | 1920 × 1080 |
| Orientation | landscape |
| Infographic style | editorial |
| Detail level | concise (minimal detail) |
| Background | `#000000` (black) |
| Text color | white (#FFFFFF) |

### 4. Use Case

This skill is designed for deep-dive educational/explanatory video covers about academic or research topics. The design philosophy ensures viewers focus on the core finding from the first second — risk/insight comes from the content itself, not decorative visuals.

Based on the **Fable 5 safety crisis** design, the core message principle is: **the risk comes not from external prompts, but from the collapse of internal task chains within the model.**
