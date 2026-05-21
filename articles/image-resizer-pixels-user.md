# How to Resize an Image by Pixels Online — Exact Width, Height, and DPI, Free

When you need an image at exactly 1200×630 pixels for an Open Graph tag, 1080×1080 for a product photo, or 1920×1080 for a banner — a general "make it smaller" slider doesn't help. You need pixel-accurate control over width and height.

The [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/) at Ultimate Tools lets you enter exact pixel dimensions, lock aspect ratio, and set DPI — all in your browser with no upload required.

---

## Set Exact Pixel Dimensions

The core use case: type the target width and height.

**Steps:**
1. Upload your image — JPG, PNG, WebP, GIF, or BMP
2. Enter target **Width** in pixels
3. Enter target **Height** in pixels
4. Click **Resize** → download

The output matches your entered pixel dimensions exactly. No approximation, no rounding.

---

## Aspect Ratio Lock

Resizing both axes independently on a non-square image gives you stretched results. Use ratio lock to avoid it:

- Enter **Width** only → enable **Lock Ratio** → height calculates automatically
- Or enter **Height** only → lock → width fills in proportionally

This is the right approach when you want to resize an image by pixels but can't risk distortion — product photos, portraits, logos.

To hit a specific ratio (16:9, 4:3, 1:1) without knowing the exact pixel targets: set one dimension, lock the ratio, and let the other fill in.

---

## Common Pixel Targets by Use Case

**Social media and web:**

| Use case | Target size |
|---|---|
| Open Graph / link preview | 1200 × 630 px |
| Twitter / X card | 1200 × 628 px |
| Instagram square | 1080 × 1080 px |
| Instagram story | 1080 × 1920 px |
| LinkedIn post | 1200 × 628 px |
| LinkedIn banner | 1584 × 396 px |
| YouTube thumbnail | 1280 × 720 px |

**Email attachments:**
Most email clients render images at 600px wide maximum. Resize to 600px wide before attaching to cut file size by 60–80% at typical camera resolutions.

**Web performance:**
A hero image from a modern phone camera is 4000×3000 pixels. Displayed at 1920px wide, you're serving 4× the data you need. Resize to exact display dimensions before uploading to your CMS.

**Thumbnails:**
Product grids, avatar images, card previews — resize to 300×300 or 400×400 to match your CSS container and avoid browser-side scaling.

---

## Reduce Image File Size by Reducing Pixel Count

Downsampling (reducing pixel dimensions) also shrinks file size. A 3000×2000 image resized to 1500×1000 is typically 65–75% smaller with no visible quality loss at normal screen viewing sizes.

The [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/) uses canvas-based bicubic sampling — the same algorithm browsers use for CSS-scaled images. Edge sharpness is preserved on most photos and graphics.

---

## DPI: When It Matters (and When It Doesn't)

DPI (dots per inch) controls print output, not screen display. On screen, only pixel dimensions matter — DPI is ignored by browsers entirely.

DPI only matters when:
- Sending to a printer or print-on-demand service
- The vendor specifies a required DPI (commonly 300 DPI for print)

**How to calculate pixel dimensions for print:**
```
Width in pixels = Print width (inches) × DPI
```

Example: 8×10 inch print at 300 DPI = 2400×3000 pixels.

Set that pixel dimension in the resizer, and embed the 300 DPI value in the file metadata. For web use, leave DPI at 72 or 96 — it has zero effect on how the image renders in a browser.

---

## Everything Runs in Your Browser

The [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/) processes your image locally using the Canvas API. Your file never leaves your device — no server upload, no account required, no watermark on the output. Download is instant.

Resize an image to exact pixel dimensions: [Image Resizer — free, pixel-accurate, no upload](https://ultimatetools.io/tools/image-tools/image-resizer/)