# Building a Browser-Based Image Rotate & Flip Tool with Canvas API

Rotating an image sounds trivial — until you need the output canvas to fit the rotated image exactly, handle arbitrary angles, and combine rotation with horizontal/vertical flipping in any order. Here's how we built the [Image Rotate tool at Ultimate Tools](https://ultimatetools.io/tools/image-tools/rotate-image/) using Canvas API, with CSS preview transforms for instant feedback and a separate canvas export path for the actual download.

---

## Two Rendering Paths

The component uses two separate rendering strategies:

**Preview** — CSS `transform` on an `<img>` tag. Instant, no canvas involved:

```tsx
const previewTransform = [
    `rotate(${normalizedDeg}deg)`,
    flipH ? "scaleX(-1)" : "",
    flipV ? "scaleY(-1)" : "",
].filter(Boolean).join(" ");

<img style={{ transform: previewTransform }} />
```

**Export** — Canvas API, runs only on download. This is where the real math happens.

Why split them? CSS transforms are free — they run on the GPU and update on every slider tick with zero lag. Canvas re-draws are heavier, especially for large images. Only computing the canvas when the user clicks Download keeps the UI snappy.

---

## The Canvas Rotation Math

The tricky part of canvas rotation is the output canvas size. A 1000×600 image rotated 45° doesn't fit in a 1000×600 canvas — the corners get clipped.

The correct output dimensions use the rotated bounding box formula:

```ts
const rad = (rotation * Math.PI) / 180;
const abscos = Math.abs(Math.cos(rad));
const abssin = Math.abs(Math.sin(rad));
const newW = Math.round(img.naturalWidth * abscos + img.naturalHeight * abssin);
const newH = Math.round(img.naturalWidth * abssin + img.naturalHeight * abscos);
```

This gives you the width and height of the axis-aligned bounding box that contains the rotated image. For 0° and 180°, it's identical to the original. For 90° and 270°, width and height swap. For 45°, it's larger than either dimension.

Then we draw by translating to the canvas centre, rotating, and drawing the image centred at the origin:

```ts
canvas.width = newW;
canvas.height = newH;
const ctx = canvas.getContext("2d")!;

ctx.save();
ctx.translate(newW / 2, newH / 2);  // move origin to canvas centre
ctx.rotate(rad);                     // rotate around that origin
ctx.drawImage(img, -img.naturalWidth / 2, -img.naturalHeight / 2);  // draw centred
ctx.restore();
```

`translate → rotate → drawImage` is the standard canvas rotation pattern. The image is drawn offset by half its dimensions so it's centred on the rotation origin.

---

## Adding Flip

Flipping uses `ctx.scale()` — negative scale values mirror the canvas along an axis:

```ts
ctx.save();
ctx.translate(newW / 2, newH / 2);
ctx.rotate(rad);
if (flipH) ctx.scale(-1, 1);   // mirror horizontally
if (flipV) ctx.scale(1, -1);   // mirror vertically
ctx.drawImage(img, -img.naturalWidth / 2, -img.naturalHeight / 2);
ctx.restore();
```

Order matters here: we apply flip **after** rotate. This means "flip the already-rotated image", which is what users expect. If you flip before rotate, you get a different result (the flip axis rotates with the image).

Both flips can be active simultaneously — `scale(-1, -1)` is a 180° point reflection, which also combines naturally with other rotations.

---

## Normalising the Rotation State

The rotation state accumulates from button presses — clicking "90° Right" three times gives `rotation = 270`, clicking again gives `360`. We normalise it before using it:

```ts
const normalizedDeg = ((rotation % 360) + 360) % 360;
```

The `+ 360) % 360` handles negative values: `-90` becomes `270`. Without this, `Math.cos(-90 * Math.PI / 180)` still works mathematically, but the bounding box calculation using `Math.abs` handles it anyway — the normalisation is mainly for display consistency.

---

## The Fine Angle Slider

The rotation presets (90°, 180°) cover common cases, but the slider allows any angle from -180° to 180°. The slider value maps to the normalised rotation:

```ts
const sliderValue = normalizedDeg > 180 ? normalizedDeg - 360 : normalizedDeg;
```

This converts the 0–360 internal representation to -180–180 for the slider range. A rotation of 270° (stored internally) displays as -90° on the slider — which matches how most people think about "rotate left 90°".

When the slider changes, it sets `rotation` directly:

```ts
onChange={(e) => setRotation(Number(e.target.value))}
```

The preset buttons accumulate on top of whatever value the slider set, because they use functional updates (`setRotation((r) => r + 90)`). Clicking "90° Right" after sliding to 15° gives 105°.

---

## Image Loading

The image is loaded once into a ref and reused for every export:

```ts
useEffect(() => {
    if (!image) return;
    const img = new window.Image();
    img.onload = () => { imgRef.current = img; };
    img.src = image;
}, [image]);
```

`imgRef.current` holds the loaded `HTMLImageElement`. The canvas export reads from this ref — no need to reload the image on every download. The `window.Image()` call is explicit to avoid SSR issues in Next.js where `Image` might be shadowed by the Next.js Image component import.

---

## Download

```ts
const ext = fileType === "image/jpeg" ? "jpg" : fileType.split("/")[1];
const link = document.createElement("a");
link.href = canvas.toDataURL(fileType, 0.95);
link.download = `${fileName}-rotated.${ext}`;
link.click();
```

We preserve the original file type. A PNG input produces a PNG output, a JPEG produces a JPEG at quality 0.95. The filename gets a `-rotated` suffix so it doesn't overwrite the original.

---

## Summary

Key patterns in this component:

- **CSS transform for preview** — GPU-accelerated, zero lag on slider input
- **Canvas only on download** — avoids heavy re-draws during interaction
- **Bounding box formula** — `w*|cos| + h*|sin|` and `w*|sin| + h*|cos|` for correct output dimensions
- **translate → rotate → scale → drawImage** — standard order for combined transforms
- **Flip after rotate** — applies mirror to the already-rotated image, matching user expectation
- **Slider maps -180 to 180**, internal state uses 0–360+ accumulation

Try the live tool: [Rotate Image](https://ultimatetools.io/tools/image-tools/rotate-image/)

This is part of [Ultimate Tools](https://ultimatetools.io/) — a free browser toolkit for PDFs, images, QR codes, and developer utilities.