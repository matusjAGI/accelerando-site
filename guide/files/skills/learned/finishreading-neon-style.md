# FinishReading Neon Style Guide

## Overview
This skill defines the brand neon glow effect for FinishReading.it. Use this when creating UI elements, icons, buttons, or animations that need our signature neon look.

## Two Modes

### 1. Subtle Mode (for background/accent elements)
Use for: triangles, icons, decorative elements, inactive states

**Characteristics:**
- Mostly solid blue fill
- Very subtle cyan rim glow
- Intensity: ~25% of full neon

**Python/Pillow Implementation:**
```python
from PIL import Image, ImageDraw, ImageFilter

# Colors
NEON_BLUE = (0, 191, 255)       # Main fill color
NEON_BLUE_GLOW = (0, 240, 255)  # Glow color (brighter cyan)

def draw_neon_polygon_subtle(img, points, intensity=0.25):
    """Subtle neon for background elements."""
    img_rgba = img.convert('RGBA')

    # Layer 1: Solid fill
    fill = Image.new('RGBA', img.size, (0, 0, 0, 0))
    ImageDraw.Draw(fill).polygon(points, fill=(*NEON_BLUE, 255))
    img_rgba = Image.alpha_composite(img_rgba, fill)

    # Layers 2-4: Subtle blurred outlines
    for radius in [10, 5, 2]:
        layer = Image.new('RGBA', img.size, (0, 0, 0, 0))
        alpha = int(255 * intensity)
        ImageDraw.Draw(layer).polygon(points, outline=(*NEON_BLUE_GLOW, alpha), width=4)
        layer = layer.filter(ImageFilter.GaussianBlur(radius=radius))
        img_rgba = Image.alpha_composite(img_rgba, layer)

    # Layer 5: Subtle rim (blended color based on intensity)
    outline = Image.new('RGBA', img.size, (0, 0, 0, 0))
    rim_color = tuple(int(NEON_BLUE[i] + (NEON_BLUE_GLOW[i] - NEON_BLUE[i]) * intensity) for i in range(3))
    ImageDraw.Draw(outline).polygon(points, outline=(*rim_color, 255), width=2)
    img_rgba = Image.alpha_composite(img_rgba, outline)

    return img_rgba.convert('RGB')
```

### 2. Pronounced Mode (for focal/interactive elements)
Use for: highlighted letters, active buttons, hover states, CTAs

**Characteristics:**
- Dark blue interior (0, 80, 160)
- Bright cyan rim/outline that glows
- Extended outer glow that fades slowly over distance
- Inner spaces (like inside 'e', 'p') fade quickly to black

**Python/Pillow Implementation:**
```python
NEON_BLUE_DARK = (0, 80, 160)   # Dark interior
NEON_BLUE_GLOW = (0, 240, 255)  # Bright glow

def draw_neon_text_pronounced(img, xy, text, font):
    """Pronounced neon for focal elements like highlighted letters."""
    img_rgba = img.convert('RGBA')

    # Layers 1-5: Outer glow (fades slowly over longer distance)
    blur_radii = [24, 16, 10, 5, 2]
    for i, radius in enumerate(blur_radii):
        layer = Image.new('RGBA', img.size, (0, 0, 0, 0))
        alpha = 220 - (i * 35)  # Gradually decrease
        ImageDraw.Draw(layer).text(xy, text, font=font, fill=(0,0,0,0),
            stroke_width=6, stroke_fill=(*NEON_BLUE_GLOW, alpha))
        layer = layer.filter(ImageFilter.GaussianBlur(radius=radius))
        img_rgba = Image.alpha_composite(img_rgba, layer)

    # Layer 6: Dark fill ON TOP (covers inner glow = fast fade inside letters)
    fill = Image.new('RGBA', img.size, (0, 0, 0, 0))
    ImageDraw.Draw(fill).text(xy, text, font=font, fill=(*NEON_BLUE_DARK, 255))
    img_rgba = Image.alpha_composite(img_rgba, fill)

    # Layer 7: Bright rim (the hot neon tube)
    stroke = Image.new('RGBA', img.size, (0, 0, 0, 0))
    bright = (min(255, NEON_BLUE_GLOW[0]+60), min(255, NEON_BLUE_GLOW[1]+60), min(255, NEON_BLUE_GLOW[2]+60))
    ImageDraw.Draw(stroke).text(xy, text, font=font, fill=(0,0,0,0),
        stroke_width=2, stroke_fill=(*bright, 255))
    img_rgba = Image.alpha_composite(img_rgba, stroke)

    return img_rgba.convert('RGB')
```

---

## Remotion/React Animations

### CSS Variables (for web/React)
```css
:root {
  --neon-blue: rgb(0, 191, 255);
  --neon-blue-dark: rgb(0, 80, 160);
  --neon-blue-glow: rgb(0, 240, 255);
  --neon-glow-subtle: 0 0 10px rgba(0, 240, 255, 0.25);
  --neon-glow-pronounced:
    0 0 5px rgba(0, 240, 255, 0.8),
    0 0 10px rgba(0, 240, 255, 0.6),
    0 0 20px rgba(0, 240, 255, 0.4),
    0 0 40px rgba(0, 240, 255, 0.2);
}
```

### Animation 1: Neon Turn On/Off (Hover)
```tsx
// NeonButton.tsx - Remotion/React component
import { interpolate, useCurrentFrame } from 'remotion';

const NeonButton = ({ children, isOn = false }) => {
  const frame = useCurrentFrame();

  // Flicker effect when turning on (like real neon)
  const flicker = isOn ? [
    { frame: 0, opacity: 0 },
    { frame: 2, opacity: 0.3 },
    { frame: 3, opacity: 0.1 },
    { frame: 5, opacity: 0.6 },
    { frame: 6, opacity: 0.2 },
    { frame: 8, opacity: 1 },
  ] : [];

  const glowIntensity = interpolate(
    frame,
    [0, 8],
    [0, 1],
    { extrapolateRight: 'clamp' }
  );

  return (
    <button
      style={{
        background: `rgb(0, ${80 + glowIntensity * 111}, ${160 + glowIntensity * 95})`,
        border: `2px solid rgb(0, ${191 + glowIntensity * 49}, 255)`,
        boxShadow: `
          0 0 ${5 * glowIntensity}px rgba(0, 240, 255, ${0.8 * glowIntensity}),
          0 0 ${10 * glowIntensity}px rgba(0, 240, 255, ${0.6 * glowIntensity}),
          0 0 ${20 * glowIntensity}px rgba(0, 240, 255, ${0.4 * glowIntensity}),
          0 0 ${40 * glowIntensity}px rgba(0, 240, 255, ${0.2 * glowIntensity})
        `,
        color: 'white',
        padding: '12px 24px',
        borderRadius: '8px',
        transition: 'all 0.3s ease',
      }}
    >
      {children}
    </button>
  );
};
```

### Animation 2: Neon Fill-Up (Progress/Attention)
```tsx
// NeonFillUp.tsx - Liquid filling animation around outline
import { interpolate, useCurrentFrame, Easing } from 'remotion';

const NeonFillUp = ({ progress = 0, children }) => {
  const frame = useCurrentFrame();

  // Progress from 0 to 1 (can be driven by scroll, time, or interaction)
  const fillProgress = interpolate(
    frame,
    [0, 60], // 2 seconds at 30fps
    [0, 1],
    { extrapolateRight: 'clamp', easing: Easing.easeInOut }
  );

  // SVG approach for precise outline animation
  return (
    <div style={{ position: 'relative' }}>
      {/* Background (dark blue interior) */}
      <div style={{
        background: 'rgb(0, 80, 160)',
        borderRadius: '8px',
        padding: '12px 24px',
      }}>
        {children}
      </div>

      {/* Animated border using conic-gradient */}
      <div style={{
        position: 'absolute',
        inset: -2,
        borderRadius: '10px',
        background: `conic-gradient(
          from 0deg,
          rgb(0, 240, 255) ${fillProgress * 360}deg,
          transparent ${fillProgress * 360}deg
        )`,
        mask: 'linear-gradient(#fff 0 0) content-box, linear-gradient(#fff 0 0)',
        maskComposite: 'xor',
        padding: '2px',
        filter: `drop-shadow(0 0 ${10 * fillProgress}px rgba(0, 240, 255, 0.6))`,
      }} />

      {/* Glow that intensifies as it fills */}
      <div style={{
        position: 'absolute',
        inset: 0,
        borderRadius: '8px',
        boxShadow: `
          0 0 ${20 * fillProgress}px rgba(0, 240, 255, ${0.4 * fillProgress}),
          0 0 ${40 * fillProgress}px rgba(0, 240, 255, ${0.2 * fillProgress})
        `,
        pointerEvents: 'none',
      }} />
    </div>
  );
};
```

### Animation 3: Neon Pulse (Attention Grabber)
```tsx
// NeonPulse.tsx - Pulsing glow effect
const NeonPulse = ({ children }) => {
  const frame = useCurrentFrame();

  // Sine wave for smooth pulsing
  const pulse = Math.sin(frame * 0.1) * 0.3 + 0.7; // 0.4 to 1.0

  return (
    <div style={{
      background: 'rgb(0, 80, 160)',
      border: '2px solid rgb(0, 240, 255)',
      borderRadius: '8px',
      padding: '12px 24px',
      boxShadow: `
        0 0 ${10 * pulse}px rgba(0, 240, 255, ${0.6 * pulse}),
        0 0 ${20 * pulse}px rgba(0, 240, 255, ${0.4 * pulse}),
        0 0 ${30 * pulse}px rgba(0, 240, 255, ${0.2 * pulse})
      `,
    }}>
      {children}
    </div>
  );
};
```

---

## Tailwind CSS Classes

```css
/* Add to tailwind.config.ts */
theme: {
  extend: {
    colors: {
      'neon-blue': 'rgb(0, 191, 255)',
      'neon-blue-dark': 'rgb(0, 80, 160)',
      'neon-blue-glow': 'rgb(0, 240, 255)',
    },
    boxShadow: {
      'neon-subtle': '0 0 10px rgba(0, 240, 255, 0.25)',
      'neon-pronounced': `
        0 0 5px rgba(0, 240, 255, 0.8),
        0 0 10px rgba(0, 240, 255, 0.6),
        0 0 20px rgba(0, 240, 255, 0.4),
        0 0 40px rgba(0, 240, 255, 0.2)
      `,
    },
  },
}

/* Usage */
<button className="bg-neon-blue-dark border-2 border-neon-blue-glow shadow-neon-subtle hover:shadow-neon-pronounced transition-shadow">
  Speed Read
</button>
```

---

## Key Principles

1. **Layer order matters**: Glow layers FIRST (blurred), then solid fill ON TOP, then sharp rim ON TOP of that
2. **Inner vs outer glow**: Fill layer covers inner glow, so spaces inside letters (e, p, o) go to black quickly while outer glow extends further
3. **Intensity control**: Use alpha values and blur radii to control intensity - subtle mode ~25%, pronounced ~100%
4. **Bright rim**: The sharp outline should be brighter than the glow (add +60 to RGB channels)
5. **Dark interior**: Pronounced mode uses darker fill (0, 80, 160) to make the rim pop more

## Color Reference
| Name | RGB | Hex | Usage |
|------|-----|-----|-------|
| Neon Blue | (0, 191, 255) | #00BFFF | Main brand blue, subtle fill |
| Neon Blue Dark | (0, 80, 160) | #0050A0 | Pronounced mode interior |
| Neon Blue Glow | (0, 240, 255) | #00F0FF | Glow/rim color |
| Bright Rim | (60, 255, 255) | #3CFFFF | Hot neon tube center |
