# FinishReading Brand & Design Guide

Complete reference for video generation, web player, and UI components.

---

## 1. Color System

### Primary: Neon Blue
```
NEON_BLUE       = rgb(0, 191, 255)   / #00BFFF  - Base blue
NEON_BLUE_DARK  = rgb(0, 80, 160)    / #0050A0  - Dark fill interior
NEON_BLUE_GLOW  = rgb(0, 240, 255)   / #00F0FF  - Bright cyan glow
```

### Background
```
BG_COLOR        = rgb(8, 8, 8)       / #080808  - Near black
BG_COLOR_DARK   = rgb(5, 5, 5)       / #050505  - Static cards
BOX_COLOR       = rgb(30, 30, 30)    / #1E1E1E  - Player outline
BOX_FILL        = rgb(12, 12, 12)    / #0C0C0C  - Player interior
```

### Text
```
TEXT_COLOR      = rgb(255, 255, 255) / #FFFFFF  - White (regular text)
TITLE_COLOR     = rgb(100, 100, 100) / #646464  - Dark gray (titles)
GRAY_400        = rgb(156, 163, 175) / #9CA3AF  - Medium gray
GRAY_500        = rgb(107, 114, 128) / #6B7280  - Muted gray
```

---

## 2. Typography

### Primary Font: SF Mono
```css
font-family: "SF Mono", "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace;
```

### Sizes
- **RSVP Word**: 84px (default), auto-scales down for long words (min 42px)
- **Title**: 20px
- **Progress text**: 16px
- **Watermark**: 14px

### Letter Spacing
```python
LETTER_SPACING = 4  # Extra pixels between letters
```

---

## 3. ORP (Optimal Recognition Point)

The ORP is where the eye naturally focuses for fastest word recognition.
Position: ~30% into the word (first third), based on cognitive research.

### Algorithm
```python
def calculate_orp_index(word: str) -> int:
    letter_count = len(re.sub(r'[^a-zA-Z]', '', word))

    if letter_count <= 1:  return 0       # 1 letter
    if letter_count <= 3:  return 0       # 1st letter
    if letter_count <= 5:  return 1       # 2nd letter (~33%)
    if letter_count <= 9:  return 2       # 3rd letter (~30%)
    if letter_count <= 13: return max(3, int(letter_count * 0.30))
    return max(4, int(letter_count * 0.30))  # 14+ letters
```

### Display Time Multipliers
```python
# Punctuation pauses
'.!?'  -> 2.0x   # Sentence endings
',;:'  -> 1.5x   # Clauses
'")—'  -> 1.25x  # Other punctuation

# Word length
1-6 letters   -> 1.0x   # Normal
7-10 letters  -> 1.2x   # Slightly longer
11-14 letters -> 1.4x   # Longer
15+ letters   -> 1.6x   # Much longer
```

---

## 4. Neon Effects

### Two Modes

**Subtle Mode** (triangles, icons) - intensity 0.25:
```css
filter: drop-shadow(0 0 5px rgba(0, 240, 255, 0.25));
```

**Pronounced Mode** (ORP letters, buttons) - full intensity:
```css
color: rgb(0, 80, 160);  /* Dark interior */
text-shadow:
  0 0 5px rgb(0, 240, 255),
  0 0 10px rgb(0, 240, 255),
  0 0 20px rgb(0, 240, 255),
  0 0 40px rgba(0, 240, 255, 0.3);
-webkit-text-stroke: 2px rgb(0, 240, 255);
```

### Python/Pillow Implementation
```python
def draw_neon_text(img, xy, text, font, fill_color, glow_color,
                   blur_radii=[24, 16, 10, 5, 2], stroke_width=2):
    img_rgba = img.convert('RGBA')

    # Layers 1-5: Blurred outer glow (fades slowly)
    for i, radius in enumerate(blur_radii):
        layer = Image.new('RGBA', img.size, (0, 0, 0, 0))
        layer_draw = ImageDraw.Draw(layer)
        alpha = 220 - (i * 35)
        layer_draw.text(xy, text, font=font, fill=(0, 0, 0, 0),
                       stroke_width=stroke_width + 4,
                       stroke_fill=(*glow_color, alpha))
        layer = layer.filter(ImageFilter.GaussianBlur(radius=radius))
        img_rgba = Image.alpha_composite(img_rgba, layer)

    # Layer 6: Solid dark fill ON TOP
    fill = Image.new('RGBA', img.size, (0, 0, 0, 0))
    fill_draw = ImageDraw.Draw(fill)
    fill_draw.text(xy, text, font=font, fill=(*fill_color, 255))
    img_rgba = Image.alpha_composite(img_rgba, fill)

    # Layer 7: Sharp bright stroke (neon rim)
    stroke = Image.new('RGBA', img.size, (0, 0, 0, 0))
    stroke_draw = ImageDraw.Draw(stroke)
    bright = (min(255, glow_color[0]+60),
              min(255, glow_color[1]+60),
              min(255, glow_color[2]+60))
    stroke_draw.text(xy, text, font=font, fill=(0, 0, 0, 0),
                    stroke_width=stroke_width,
                    stroke_fill=(*bright, 255))
    img_rgba = Image.alpha_composite(img_rgba, stroke)

    return img_rgba.convert('RGB')
```

### Key Insight
**Layer order is critical:**
1. Outer glow (blurred) - FIRST (background)
2. Solid dark fill - ON TOP (covers inner glow, makes interior dark)
3. Sharp bright stroke - ON TOP (the hot neon rim)

---

## 5. Player Layout

### Dimensions
```python
FRAME_WIDTH = 800
FRAME_HEIGHT = 400
box_margin = 20      # From edges (narrow for long words)
box_height = 260     # Player box height
corner_radius = 15   # Rounded corners
```

### Elements
- **Title**: Top, dark gray, truncated if too long
- **Player box**: Rounded rectangle with subtle fill
- **Triangles**: Above and below center, pointing at ORP, subtle neon
- **Horizontal lines**: From box edges toward center, with gap
- **Progress bar**: Bottom, 4px height, neon blue fill
- **Word count**: Below progress bar

### Triangle Positioning
```python
triangle_offset = 100    # Distance from center to line
triangle_size = 14       # Size of triangle
line_gap = 35           # Gap near center where lines stop
```

---

## 6. Auto Font Scaling

For long words that would overflow:
```python
max_word_width = (FRAME_WIDTH - box_margin * 2) - 40
test_width = measure_word_with_spacing(word, font)

if test_width > max_word_width:
    scale = max_word_width / test_width
    actual_font_size = max(42, int(font_size * scale * 0.88))  # 12% margin
```

---

## 7. Animations (Web/React)

### Hover: Dark → Light Flip
```tsx
style={{
  backgroundColor: hovered ? colors.base : colors.dark,
  border: `2px solid ${hovered ? colors.glow : colors.base}`,
  boxShadow: hovered ? glowPronounced : glowSubtle,
  color: hovered ? '#000' : '#fff',
}}
```

### Flicker On
```tsx
const flickerPattern = [0.3, 0.1, 0.6, 0.2, 0.8, 0.4, 1, 1];
// 80ms between steps
```

### Fill-Up
```tsx
// 38ms interval (30% faster than 50ms)
// Glow appears with fill progress
filter: `drop-shadow(0 0 ${12 * progress}px rgba(r,g,b, ${0.72 * progress}))`;
```

### Pulse
```tsx
const intensity = Math.sin(phase) * 0.3 + 0.7;  // Range 0.4 - 1.0
// 50ms interval for phase update
```

---

## 8. Files Reference

### Video Generation
- `bot/gif_generator/frames.py` - Frame rendering, neon effects
- `bot/gif_generator/orp.py` - ORP algorithm, pause multipliers
- `bot/gif_generator/assembler.py` - Video assembly

### Web Player
- `web/src/components/reader/RSVPPlayer.tsx` - Main player
- `web/src/app/read/[id]/page.tsx` - Full-view mode
- `web/src/app/neon-test/page.tsx` - Interactive demo

### Constants
- `web/src/lib/constants.ts` - Add neon colors here

---

## 9. Quick Reference

| Element | Mode | Color | Effect |
|---------|------|-------|--------|
| ORP letter | Pronounced | BLUE_DARK fill, BLUE_GLOW stroke | Full glow |
| Regular text | None | WHITE | No effect |
| Triangles | Subtle | BLUE base | 0.25 intensity glow |
| Progress bar | None | BLUE | Solid fill |
| Buttons (hover) | Flip | DARK → BASE | Glow intensifies |

---

## 10. Testing

Live demo: `http://localhost:3001/neon-test`

Video test:
```bash
python3 scripts/preview-video.py --demo
open preview_output.mp4
```
