# Building a Browser-Based Image Resizer with Canvas API and Aspect Ratio Lock

Resizing images in the browser is straightforward with Canvas API — but there are a few details worth getting right: aspect ratio calculation on linked inputs, `imageSmoothingQuality` for better output, and loading the image at download time rather than on upload. Here's how we built the [Image Resizer at Ultimate Tools](https://ultimatetools.io/tools/image-tools/image-resizer/).

---

## State Design

```ts
const [originalDimensions, setOriginalDimensions] = useState({ width: 0, height: 0 });
const [width, setWidth] = useState(0);
const [height, setHeight] = useState(0);
const [quality, setQuality] = useState(80);
const [maintainAspectRatio, setMaintainAspectRatio] = useState(true);
const [fileType, setFileType] = useState("image/jpeg");
```

`originalDimensions` is immutable after load — it's the reference for aspect ratio calculations. `width` and `height` are the working values the user edits. Keeping them separate means you can always reset to original without re-reading the file.

---

## Loading the Image

The file is read with `FileReader` and loaded into an `Image` element to get the natural dimensions:

```ts
const processFile = (file: File) => {
    if (!file.type.startsWith("image/")) return;

    setFileType(file.type);
    setFileName(file.name.split('.')[0]);

    const reader = new FileReader();
    reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
            setOriginalDimensions({ width: img.width, height: img.height });
            setWidth(img.width);
            setHeight(img.height);
            setImage(e.target?.result as string);
        };
        img.src = e.target?.result as string;
    };
    reader.readAsDataURL(file);
};
```

The nested callbacks — `FileReader.onload` → `Image.onload` — are necessary because you can't get image dimensions from a `File` object directly. You need to decode the image first. The data URL is stored in state for the preview `<img>` and reused at download time.

---

## Aspect Ratio Lock

When aspect ratio lock is on, changing width recalculates height proportionally, and vice versa:

```ts
const handleWidthChange = (val: number) => {
    setWidth(val);
    if (maintainAspectRatio && originalDimensions.width > 0) {
        const newHeight = Math.round((val / originalDimensions.width) * originalDimensions.height);
        setHeight(newHeight);
    }
};

const handleHeightChange = (val: number) => {
    setHeight(val);
    if (maintainAspectRatio && originalDimensions.height > 0) {
        const newWidth = Math.round((val / originalDimensions.height) * originalDimensions.width);
        setWidth(newWidth);
    }
};
```

The ratio is always calculated from the **original** dimensions, not the current ones. If you calculated from the current value, floating point rounding in successive edits would gradually drift the ratio. Using the original as the fixed reference keeps it accurate regardless of how many times the user changes the values.

`Math.round` prevents fractional pixel values — canvas dimensions must be integers.

---

## Canvas Resize and Download

The canvas is hidden in the DOM and only written to on download:

```ts
const downloadImage = () => {
    const canvas = canvasRef.current;
    if (!canvas || !image) return;

    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    const img = new Image();
    img.onload = () => {
        canvas.width = width;
        canvas.height = height;

        ctx.imageSmoothingEnabled = true;
        ctx.imageSmoothingQuality = "high";

        ctx.drawImage(img, 0, 0, width, height);

        const dataUrl = canvas.toDataURL(fileType, quality / 100);
        const link = document.createElement("a");
        link.download = `${fileName}-resized.${fileType.split('/')[1]}`;
        link.href = dataUrl;
        link.click();
    };
    img.src = image;
};
```

### imageSmoothingQuality

`ctx.imageSmoothingQuality = "high"` enables bilinear or bicubic interpolation (browser-dependent) instead of nearest-neighbour. For downscaling photos this makes a visible difference — edges stay smooth rather than pixelated. It's a one-liner with a meaningful quality impact.

### Why reload the image at download time?

The `img.src = image` inside `downloadImage` reloads the image from the stored data URL. An alternative would be to keep a persistent `imgRef` loaded on upload. Both work, but reloading on download avoids holding a decoded bitmap in memory throughout the session — for large images this matters on low-memory devices.

### Quality parameter

`quality / 100` maps the 1–100 slider to the 0–1 range `toDataURL` expects. This only affects JPEG and WebP output — PNG ignores the quality argument since it's lossless.

---

## Preview with CSS aspect-ratio

The preview `<img>` uses the original aspect ratio to prevent layout shift as the image loads:

```tsx
<img
    src={image}
    alt="Preview"
    style={{
        aspectRatio: `${originalDimensions.width}/${originalDimensions.height}`
    }}
/>
```

This reserves the correct space before the image renders. Without it, the layout jumps when the image loads — especially noticeable on slower connections or large files.

---

## Reset

```ts
const reset = () => {
    setWidth(originalDimensions.width);
    setHeight(originalDimensions.height);
    setQuality(80);
};
```

Restores to original dimensions and default quality. Because `originalDimensions` is never mutated after load, reset is always correct regardless of how many edits the user made.

---

## Summary

Key decisions in this component:

- **Original dimensions as fixed reference** for aspect ratio — prevents float drift on repeated edits
- **`Math.round` on calculated dimension** — canvas requires integer dimensions
- **`imageSmoothingQuality = "high"`** — better interpolation for downscaling, one line
- **Reload image at download time** — avoids holding a decoded bitmap in memory during editing
- **CSS `aspect-ratio` on preview** — prevents layout shift before image renders
- **Hidden canvas in DOM** — written only on download, not on every dimension change

Try the live tool: [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/)

This is part of [Ultimate Tools](https://ultimatetools.io/) — a free browser toolkit for PDFs, images, QR codes, and developer utilities.