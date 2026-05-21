# How to Blur Part of an Image Free — Blur a Face, License Plate, or Specific Area Without an App

You have a photo with a face in it. Or a license plate. Or an account number visible in a screenshot. You need to blur just that part — not the whole image.

Most free tools only blur the entire image. Fotor and BeFunky put selective (regional) blur behind paid plans.

The [Image Blur Tool](https://ultimatetools.io/tools/image-tools/blur-image/) at Ultimate Tools adds a **Select Area** mode free. Draw a rectangle over any region, set the blur intensity, and download. Add as many regions as you need — all in one pass.

---

## How to Blur Only Part of an Image

1. Open the [free selective blur tool](https://ultimatetools.io/tools/image-tools/blur-image/)
2. Upload your photo (JPG, PNG, WebP supported)
3. Click the **Select Area** tab
4. Click and drag on the image to draw a rectangle over the area to blur
5. Add more rectangles as needed (face, plate, address — all in one pass)
6. Adjust blur intensity: Subtle (3px), Medium (10px), Heavy (25px), or custom slider up to 50px
7. Click **Download Image**

Each region is listed below the image with its dimensions and a delete button. You can remove individual regions and re-draw them.

---

## How It Works Technically

The tool uses the browser's Canvas API. For each defined region:

1. The original image is drawn on a full-size canvas
2. A clipping rectangle is applied to that region
3. `ctx.filter = "blur(Xpx)"` is set and the image is redrawn *only inside the clip* — the rest stays sharp
4. The clip is released and the next region is processed

This means the blur is applied pixel-accurate to the selected rectangle. No blur bleeds outside the region because the canvas clip prevents it.

All processing runs locally in your browser. The image is never uploaded to any server.

---

## How Strong Should the Blur Be?

| Use case | Recommended blur | Why |
|---|---|---|
| Face for privacy | 20–25px (Heavy) | Facial features become unrecognizable |
| License plate | 15–25px | Numbers become unreadable |
| Account number / address | 10–15px | Text becomes illegible |
| Background element | 5–10px | Visible but not distracting |

For legal or journalistic use where identity must be fully protected, use 25–50px. At 25px, individual features are no longer distinguishable.

---

## Multiple Regions — All in One Download

Each drag creates a new numbered region. You can blur a face, a license plate, and a visible address all at once:

- Region 1 → face (top-left of image)
- Region 2 → license plate (bottom-right)
- Region 3 → name visible in the background

All three blur regions are applied before you download, so the final image contains all three blurred areas in a single file. No need to run the tool multiple times.

---

## Compared to Paid Tools

| Tool | Selective blur | Price |
|---|---|---|
| Fotor | Yes | Paid plan |
| BeFunky | Yes | Paid plan |
| Canva | Limited (shape mask) | Paid plan |
| **Ultimate Tools** | **Yes — unlimited regions** | **Free** |

---

No signup, no watermark, no export limit. The image never leaves your browser.

---

## Related Tools

- [batch compress multiple images free download as ZIP](https://ultimatetools.io/tools/image-tools/image-compressor/) — compress the blurred image before emailing or uploading — batch mode handles multiple files at once with no upload limit
- [batch watermark multiple images free](https://ultimatetools.io/tools/image-tools/watermark-image/) — add a copyright or brand watermark after blurring — upload multiple photos and download all watermarked files as a ZIP
- [crop image online free no signup](https://ultimatetools.io/tools/image-tools/crop-image/) — crop the image to remove surrounding context after blurring a region — custom dimensions or preset aspect ratios

---

**[Blur part of an image for free →](https://ultimatetools.io/tools/image-tools/blur-image/)**