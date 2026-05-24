# Hand-Drawn QR Codes with Canvas — Whole-Image Stroke Field

A hand-drawn QR code on graph paper recently went viral on Reddit — 4,000+ upvotes for a single image of someone painstakingly drawing one by hand, cell by cell, in pencil. The comments were full of two questions: does it actually scan, and is there a way to generate one programmatically that still looks hand-drawn?

This article walks through the algorithm I built for the [hand-drawn QR code generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) at Ultimate Tools. The first three approaches all failed in the same way — every per-cell rendering technique produces output that reads as a barcode-style grid of patches, not a human-drawn QR. The fix is a different approach entirely: a **whole-image stroke field** that renders ink only where it crosses dark cells.

---

## Why Per-Cell Rendering Always Fails for Hand-Drawn QR Codes

The natural starting point is to iterate over the QR matrix, find each dark cell, and draw N sketched strokes inside it. The library exposes the data cleanly:

```typescript
import qrcode from "qrcode-generator";

const qr = qrcode(0, "H");
qr.addData(data);
qr.make();
const moduleCount = qr.getModuleCount();

for (let row = 0; row < moduleCount; row++) {
  for (let col = 0; col < moduleCount; col++) {
    if (qr.isDark(row, col)) {
      drawSketchedCell(ctx, col * cellSize, row * cellSize, cellSize);
    }
  }
}
```

The `drawSketchedCell` function can do anything — random angles per cell, varied stroke counts, pressure variation, cross-hatching. I tried all of it. **The result always looks like a grid of small barcode patches.**

The reason is structural. When you confine strokes inside a cell, every stroke starts and ends at a cell boundary. Adjacent dark cells produce visually distinct stroke clusters with sharp gaps between them — the eye reads "computer drew this one cell at a time", not "a human swept a pencil across the page."

You can add random angle variation per cell, but it doesn't help. The barcode effect comes from stroke termination at cell edges, not from angle uniformity.

---

## The Whole-Image Stroke Field

A human drawing this doesn't lift the pencil between adjacent dark cells. They sweep through groups. Long strokes naturally span multiple connected dark cells and break only at light gaps.

The algorithm that produces this output works in reverse of the per-cell approach. Instead of "for each dark cell, draw strokes inside it," we do "for each long stroke across the entire image, walk along it pixel by pixel, draw ink only where it crosses a dark cell."

```typescript
function drawStrokeLayer(
  ctx: CanvasRenderingContext2D,
  qr: QRCode,
  moduleCount: number,
  offset: number,
  cellSize: number,
  angleDeg: number,
  layerStrength: number,
  colorStr: string
) {
  const angle = (angleDeg * Math.PI) / 180;
  const dx = Math.cos(angle);
  const dy = Math.sin(angle);
  const perpDx = -dy;
  const perpDy = dx;

  const totalSize = moduleCount * cellSize;
  const centerX = offset + totalSize / 2;
  const centerY = offset + totalSize / 2;
  const radius = (totalSize * Math.SQRT2) / 2 + cellSize * 2;

  const strokeSpacing = Math.max(1.5, cellSize / 6);
  const numStrokes = Math.ceil((2 * radius) / strokeSpacing);

  for (let s = 0; s < numStrokes; s++) {
    const sOffset = (s - numStrokes / 2 + 0.5) * strokeSpacing
                  + (Math.random() - 0.5) * strokeSpacing * 0.3;
    const lineCenterX = centerX + perpDx * sOffset;
    const lineCenterY = centerY + perpDy * sOffset;

    let inRun = false;
    let runStartX = 0;
    let runStartY = 0;

    for (let t = -radius; t <= radius; t += 1) {
      const x = lineCenterX + dx * t;
      const y = lineCenterY + dy * t;
      const col = Math.floor((x - offset) / cellSize);
      const row = Math.floor((y - offset) / cellSize);
      const inside = col >= 0 && col < moduleCount && row >= 0 && row < moduleCount;
      const isDarkHere = inside && qr.isDark(row, col);

      if (isDarkHere && !inRun) {
        inRun = true;
        runStartX = x;
        runStartY = y;
      } else if (!isDarkHere && inRun) {
        inRun = false;
        drawStrokeSegment(ctx, runStartX, runStartY, x, y, layerStrength, colorStr);
      }
    }
  }
}
```

Each stroke is a single straight line that crosses the entire image at a given angle. The algorithm walks along it in 1-pixel steps. When it enters a dark cell, it remembers the entry point. When it exits into a light cell, it draws a stroke segment between those two points.

The crucial property: **a single stroke that crosses through four adjacent dark cells produces one continuous line spanning all four**. The same stroke crossing a checkerboard pattern produces a broken series of short segments. Nothing is drawn over light areas.

This is exactly what a human pencil sweep does.

---

## Two Layers, Random Angles

A single layer of parallel strokes still looks too regular — every cell is filled with strokes going the same direction. The fix is two layers at different angles:

```typescript
const angle1 = Math.random() * 60 - 30;        // -30° to +30°
const angle2 = angle1 + 60 + Math.random() * 30; // 60° to 90° offset

drawStrokeLayer(ctx, qr, moduleCount, offset, cellSize, angle1, 1.0, colorStr);
drawStrokeLayer(ctx, qr, moduleCount, offset, cellSize, angle2, 0.85, colorStr);
```

Two layers cross-hatched produce the woven texture of real pencil shading. The 60–90° offset is wide enough to look intentional and narrow enough to look organic — perfect-perpendicular crosses (exactly 90°) look mechanical.

The second layer renders at 85% strength so the first layer dominates visually. This matches how hand-shading actually works: the dominant stroke direction is laid down first, the cross direction is lighter.

---

## Tuning for Scan Reliability

The trade-off in every iteration was the same: looks more hand-drawn (sparser strokes, visible paper between strokes) versus stays scannable (denser ink, cells read as solid dark to a camera).

The final values that pass both tests:

```typescript
const strokeSpacing = cellSize / 6;     // 6 strokes per cell width per layer
const baseLineWidth = cellSize / 6.5;   // proportional to cell size
const opacity = 0.82 + Math.random() * 0.18; // 82–100%
const extension = cellSize * 0.35;       // strokes extend 35% past cell boundary
```

Two layers × six strokes per cell width = twelve strokes through each dark cell on average. With strokes at 82–100% opacity, the cell's average darkness is ~75–85% — comfortably above the threshold any QR scanner uses to classify a cell as dark.

The `extension` parameter is what makes cell boundaries look ragged. Each stroke starts slightly before it enters a dark cell and ends slightly after it exits — randomized per stroke. Without this, the rendered output has the same pixel-perfect cell boundaries as the source, defeating the whole point.

---

## Pitfalls Worth Knowing

A few things that surprised me during five iterations:

- **Anti-aliasing eats darkness at higher resolutions.** At 1024px output, each stroke's edge pixels are gray due to canvas anti-aliasing. The cell averages darker than 512px output because of this — counterintuitively, you need *higher* minimum opacity at higher resolutions to compensate.

- **Short content matters more than you think.** A QR encoding a long URL produces a higher-version QR with smaller cells. Smaller cells mean thinner strokes (since stroke width is proportional to cell size), which means hand-drawn texture becomes invisible. For social media pin-style use cases, encode short URLs.

- **`lineCap: "round"` is doing real work.** With `lineCap: "butt"` (the default), stroke endpoints look like blunt rectangles. With round caps, the endpoints taper like a real pencil tip. Tiny visual change, massive aesthetic improvement.

---

## See It Running

The full implementation is in [lib/sketchQrEffect.ts](https://github.com/Shaishavap/ultimatetools-articles) on GitHub. Try the live tool to generate a hand-drawn QR for a URL of your choice — toggle the **Pencil Sketch Style** option in the design panel.

---

## Related Tools

- [scan QR code online from browser free](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/) — verify your hand-drawn QR decodes correctly before printing — upload the generated PNG or use your webcam to confirm the content
- [bulk QR code generator from CSV free](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) — generate hundreds of QR codes from a spreadsheet for event tickets or product tags
- [compress PNG image free no quality loss](https://ultimatetools.io/tools/image-tools/image-compressor/) — reduce the hand-drawn QR PNG before embedding in print files or web pages

---

The whole-image stroke field is a reusable pattern. Anywhere you have a binary grid and want output that looks human-drawn — Wordle screenshots, Sudoku boards, pixel art — the same approach works: render long parallel strokes across the entire image, mask them against the grid, let the strokes break naturally at boundaries.

**[Generate your own free hand-drawn QR code with pencil sketch style →](https://ultimatetools.io/tools/misc-tools/qr-code-generator/)**