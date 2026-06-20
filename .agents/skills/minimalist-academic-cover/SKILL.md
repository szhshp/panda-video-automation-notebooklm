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

Run the following Python script to generate a 1920×1080 pure black cover with bright yellow centered titles and dark stroke outline. Customize `title`/`caption` for the topic.

**Design rules:**
- Font sizes: title 130px, subtitle 86px (+10 over default)
- Titles: bright yellow (#FFD700) with dark brown stroke (width 3)
- Caption: light gray with stroke (width 2)
- All content must fit within the **center 60%** of the canvas (y: 216-864)
- If caption is too long, **split into two lines** to prevent canvas overflow

```python
from PIL import Image, ImageDraw, ImageFont

W, H = 1920, 1080
img = Image.new('RGB', (W, H), color=(0, 0, 0))
draw = ImageDraw.Draw(img)

# Chinese-capable font — adjust path per OS
font_path = '/System/Library/Fonts/STHeiti Light.ttc'  # macOS

title1 = '[video-title-line-1]'
title2 = '[video-title-line-2]'
cap1 = '[caption-line-1]'
cap2 = '[caption-line-2 (split if too long)]'

font_title = ImageFont.truetype(font_path, 130)
font_sub = ImageFont.truetype(font_path, 86)

BRIGHT_YELLOW = (255, 215, 0)   # #FFD700
LIGHT_GRAY = (200, 200, 200)
STROKE_COLOR = (30, 20, 0)      # dark brown outline

# Compute text bboxes
b1 = draw.textbbox((0, 0), title1, font=font_title)
tw1, th1 = b1[2] - b1[0], b1[3] - b1[1]
b2 = draw.textbbox((0, 0), title2, font=font_title)
tw2, th2 = b2[2] - b2[0], b2[3] - b2[1]
b3 = draw.textbbox((0, 0), cap1, font=font_sub)
tw3, th3 = b3[2] - b3[0], b3[3] - b3[1]
b4 = draw.textbbox((0, 0), cap2, font=font_sub)
tw4, th4 = b4[2] - b4[0], b4[3] - b4[1]

gap_1_2 = 80
gap_2_3 = 140
gap_cap = 15

total_height = th1 + gap_1_2 + th2 + gap_2_3 + th3 + gap_cap + th4
block_top = (H - total_height) // 2

# Title line 1 — centered, bright yellow with stroke
tx1, ty1 = (W - tw1) // 2, block_top
draw.text((tx1, ty1), title1, fill=BRIGHT_YELLOW, font=font_title,
          stroke_width=3, stroke_fill=STROKE_COLOR)

# Title line 2 — below with gap, same style
tx2, ty2 = (W - tw2) // 2, ty1 + th1 + gap_1_2
draw.text((tx2, ty2), title2, fill=BRIGHT_YELLOW, font=font_title,
          stroke_width=3, stroke_fill=STROKE_COLOR)

# Caption line 1 — light gray with stroke
tx3, ty3 = (W - tw3) // 2, ty2 + th2 + gap_2_3
draw.text((tx3, ty3), cap1, fill=LIGHT_GRAY, font=font_sub,
          stroke_width=2, stroke_fill=STROKE_COLOR)

# Caption line 2 — below caption line 1
tx4, ty4 = (W - tw4) // 2, ty3 + th3 + gap_cap
draw.text((tx4, ty4), cap2, fill=LIGHT_GRAY, font=font_sub,
          stroke_width=2, stroke_fill=STROKE_COLOR)

img.save('input/cover.png')
```

**Adjust font sizes** by changing `ImageFont.truetype()` arguments (title 130, subtitle 86 recommended). **Adjust vertical spacing** by changing `gap_1_2`, `gap_2_3`, `gap_cap`. **If caption fits in one line**, omit `cap2` and adjust total_height accordingly.

### Step 3: Copy to Output Path

```bash
cp input/cover.png output/video/cover.png
```

### Step 4: Verify Output

Confirm the output file exists at `input/cover.png` and is a valid PNG (check it visually).

---

## Design Specification

### 1. Core Visual Elements

- **Background** — MUST be **solid black (#000000)**. No gradients, textures, decorative lines, or background images. This ensures absolute visual focus.
- **Main Title** — Extra Large font size (130px), **Bold** weight, Sans-serif. Centered horizontal alignment. **Bright yellow (#FFD700)** with dark brown outline stroke (width 3).
- **Subtitle** — Same style as main title (bright yellow #FFD700, stroke width 3). Placed below main title with gap (~80px).
- **Caption** — Font size **86px**. **Light gray (#C8C8C8)** with outline stroke (width 2). Placed below subtitle. Split into two lines if content is too long.
- **Content constraint** — All text elements must fit within the **center 60%** of the canvas (y: 216-864) to ensure visual composition is tight and centered.
- **Strictly forbidden** — graphics, icons, illustrations, decorative lines, dates, author names, logos, watermarks. Only title + subtitle + caption exist on the canvas.

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
| Title color | bright yellow `#FFD700` with stroke |
| Subtitle color | bright yellow `#FFD700` with stroke |
| Caption color | light gray `#C8C8C8` with stroke |
| Content area | center 60% (y: 216-864) |

### 4. Use Case

This skill is designed for deep-dive educational/explanatory video covers about academic or research topics. The design philosophy ensures viewers focus on the core finding from the first second — risk/insight comes from the content itself, not decorative visuals.

Based on the **Fable 5 safety crisis** design, the core message principle is: **the risk comes not from external prompts, but from the collapse of internal task chains within the model.**
