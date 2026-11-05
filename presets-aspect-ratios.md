# Preset boxes by app and aspect ratio

Where common AI video generators and social apps put their visible mark, as starting boxes for a client-side remover. These are the presets used by [Free Video Watermark Remover](https://freevideowatermarkremover.com) (source: [presets.ts](https://freevideowatermarkremover.com/engine/presets.ts.txt)).

**They are starting points, not measurements of every export.** Apps move and resize their badges between versions and plans. Always show the box on the user's own frame and let them drag and resize it before cleaning.

## Conventions

- Boxes are fractions of the frame: `x`, `y` from the top-left corner, `w`, `h` as a share of width and height. Fractions survive any resolution, so a single preset covers 720p, 1080p and 4K.
- The aspect group comes from `width / height`: above 1.15 is **landscape (16:9)**, below 0.87 is **portrait (9:16)**, anything in between is **square (1:1)**.
- Pixel values below are at the reference size shown; multiply the fractions by your own frame size.
- Before filling, pad each box by 2 px on every side so the anti-aliased edge of the mark is rebuilt too.
- "Share rebuilt" is the total area of the preset's boxes as a share of the frame. Everything else is left exactly as decoded.

```ts
type Shape = 'landscape' | 'portrait' | 'square';
function shapeOf(width: number, height: number): Shape {
  const ratio = width / height;
  if (ratio > 1.15) return 'landscape';
  if (ratio < 0.87) return 'portrait';
  return 'square';
}
function toPixels(box, width, height, pad = 2) {
  const x = Math.max(0, Math.floor(box.x * width) - pad);
  const y = Math.max(0, Math.floor(box.y * height) - pad);
  const r = Math.min(width, Math.ceil((box.x + box.w) * width) + pad);
  const b = Math.min(height, Math.ceil((box.y + box.h) * height) + pad);
  return { x, y, w: r - x, h: b - y };
}
```

## Presets

### Seedance 2.0 / Dreamina

| Aspect | Reference size | Box | Fraction of frame (x, y, w, h) | Pixels at reference size (x, y, w × h) | Share rebuilt |
|---|---|---|---|---|---|
| 16:9 | 1920×1080 | 1 | 0.8, 0.88, 0.18, 0.09 | 1536, 950, 346 × 97 | 1.6% |
| 9:16 | 1080×1920 | 1 | 0.64, 0.915, 0.33, 0.06 | 691, 1757, 356 × 115 | 2.0% |
| 1:1 | 1080×1080 | 1 | 0.74, 0.895, 0.24, 0.08 | 799, 967, 259 × 86 | 1.9% |

### Higgsfield AI

| Aspect | Reference size | Box | Fraction of frame (x, y, w, h) | Pixels at reference size (x, y, w × h) | Share rebuilt |
|---|---|---|---|---|---|
| 16:9 | 1920×1080 | 1 | 0.79, 0.865, 0.19, 0.1 | 1517, 934, 365 × 108 | 1.9% |
| 9:16 | 1080×1920 | 1 | 0.62, 0.905, 0.35, 0.065 | 670, 1738, 378 × 125 | 2.3% |
| 1:1 | 1080×1080 | 1 | 0.73, 0.885, 0.25, 0.09 | 788, 956, 270 × 97 | 2.2% |

### Sora 2

| Aspect | Reference size | Box | Fraction of frame (x, y, w, h) | Pixels at reference size (x, y, w × h) | Share rebuilt |
|---|---|---|---|---|---|
| 16:9 | 1920×1080 | 1 | 0.02, 0.42, 0.17, 0.12 | 38, 454, 326 × 130 | 4.2% |
| 16:9 | 1920×1080 | 2 | 0.8, 0.06, 0.18, 0.12 | 1536, 65, 346 × 130 | 4.2% |
| 9:16 | 1080×1920 | 1 | 0.03, 0.44, 0.34, 0.07 | 32, 845, 367 × 134 | 4.8% |
| 9:16 | 1080×1920 | 2 | 0.62, 0.07, 0.35, 0.07 | 670, 134, 378 × 134 | 4.8% |
| 1:1 | 1080×1080 | 1 | 0.02, 0.43, 0.24, 0.1 | 22, 464, 259 × 108 | 4.9% |
| 1:1 | 1080×1080 | 2 | 0.73, 0.06, 0.25, 0.1 | 788, 65, 270 × 108 | 4.9% |

### PixVerse

| Aspect | Reference size | Box | Fraction of frame (x, y, w, h) | Pixels at reference size (x, y, w × h) | Share rebuilt |
|---|---|---|---|---|---|
| 16:9 | 1920×1080 | 1 | 0.81, 0.875, 0.17, 0.09 | 1555, 945, 326 × 97 | 1.5% |
| 9:16 | 1080×1920 | 1 | 0.66, 0.915, 0.31, 0.06 | 713, 1757, 335 × 115 | 1.9% |
| 1:1 | 1080×1080 | 1 | 0.75, 0.89, 0.23, 0.08 | 810, 961, 248 × 86 | 1.8% |

### Hailuo / MiniMax

| Aspect | Reference size | Box | Fraction of frame (x, y, w, h) | Pixels at reference size (x, y, w × h) | Share rebuilt |
|---|---|---|---|---|---|
| 16:9 | 1920×1080 | 1 | 0.8, 0.87, 0.18, 0.1 | 1536, 940, 346 × 108 | 1.8% |
| 9:16 | 1080×1920 | 1 | 0.63, 0.91, 0.34, 0.065 | 680, 1747, 367 × 125 | 2.2% |
| 1:1 | 1080×1080 | 1 | 0.74, 0.885, 0.24, 0.09 | 799, 956, 259 × 97 | 2.2% |

### TikTok / Reels saved clip

| Aspect | Reference size | Box | Fraction of frame (x, y, w, h) | Pixels at reference size (x, y, w × h) | Share rebuilt |
|---|---|---|---|---|---|
| 16:9 | 1920×1080 | 1 | 0.02, 0.04, 0.22, 0.13 | 38, 43, 422 × 140 | 6.1% |
| 16:9 | 1920×1080 | 2 | 0.75, 0.82, 0.23, 0.14 | 1440, 886, 442 × 151 | 6.1% |
| 9:16 | 1080×1920 | 1 | 0.03, 0.035, 0.4, 0.085 | 32, 67, 432 × 163 | 7.2% |
| 9:16 | 1080×1920 | 2 | 0.55, 0.865, 0.42, 0.09 | 594, 1661, 454 × 173 | 7.2% |
| 1:1 | 1080×1080 | 1 | 0.03, 0.035, 0.3, 0.1 | 32, 38, 324 × 108 | 6.1% |
| 1:1 | 1080×1080 | 2 | 0.66, 0.86, 0.31, 0.1 | 713, 929, 335 × 108 | 6.1% |

### Custom (default box)

| Aspect | Reference size | Box | Fraction of frame (x, y, w, h) | Pixels at reference size (x, y, w × h) | Share rebuilt |
|---|---|---|---|---|---|
| 16:9 | 1920×1080 | 1 | 0.76, 0.84, 0.2, 0.1 | 1459, 907, 384 × 108 | 2.0% |
| 9:16 | 1080×1920 | 1 | 0.6, 0.88, 0.34, 0.07 | 648, 1690, 367 × 134 | 2.4% |
| 1:1 | 1080×1080 | 1 | 0.7, 0.86, 0.26, 0.09 | 756, 929, 281 × 97 | 2.3% |



## Notes per app

- **Seedance 2.0 / Dreamina.** One mark in the bottom-right corner; Dreamina exports use the same spot. If the user's plan offers watermark-free downloads, re-downloading is always cleaner than rebuilding. Tool page: [freevideowatermarkremover.com/seedance-watermark-remover](https://freevideowatermarkremover.com/seedance-watermark-remover)
- **Higgsfield AI.** One bottom-right badge. Fast camera-motion presets carry texture through the box, so keep it tight. Tool page: [/higgsfield-watermark-remover](https://freevideowatermarkremover.com/higgsfield-watermark-remover)
- **Sora 2.** The visible mark moves during a clip, so the preset starts with two boxes on the spots it visits most. Scrub the whole clip and add or move boxes until every position is covered. Tool page: [freevideowatermarkremover.com/sora-2-watermark-remover](https://freevideowatermarkremover.com/sora-2-watermark-remover)
- **PixVerse.** One bottom-right logo; widen the box when an export carries a longer text mark. Tool page: [/pixverse-watermark-remover](https://freevideowatermarkremover.com/pixverse-watermark-remover)
- **Hailuo / MiniMax.** One bottom-right mark. Tool page: [/hailuo-watermark-remover](https://freevideowatermarkremover.com/hailuo-watermark-remover)
- **TikTok / Reels saved clip.** Two boxes: the logo and @handle start top-left, then jump to the bottom-right later in the clip. Tool page: [/remove-tiktok-watermark-from-video](https://freevideowatermarkremover.com/remove-tiktok-watermark-from-video)
- **Custom.** A default box near the bottom-right, meant to be dragged anywhere.

## Tips that matter more than the coordinates

1. Keep boxes tight. Every pixel inside is rebuilt rather than kept.
2. Scrub before cleaning. A mark that moves needs a box on each spot.
3. If an edge of the mark still shows, grow the box by a few pixels and redo.
4. Clean corners over sky, water, walls and soft backgrounds come out best; detail right behind the mark will look smoothed.

All brand names are trademarks of their owners. These notes are independent and not affiliated with any of them.
