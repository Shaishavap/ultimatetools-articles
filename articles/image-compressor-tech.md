# Building a Browser-Side Image Compressor with Canvas API — Quality Presets, WebP Conversion, and ZIP Download

Image compression is usually a server job — you upload, the server runs ImageMagick or Sharp, sends back a smaller file. But the Canvas API's `toDataURL` method does the same thing in the browser with one line. Here's how the [Image Compressor](https://ultimatetools.io/tools/image-tools/image-compressor/) is built: no server, no upload, batch processing with parallel compression, and ZIP download for multiple files.

## The Core: `canvas.toDataURL`

The entire compression pipeline is one Canvas operation:

```typescript
const canvas = document.createElement("canvas");
canvas.width = targetW;
canvas.height = targetH;
const ctx = canvas.getContext("2d")!;
ctx.drawImage(img, 0, 0, targetW, targetH);

const outputMime = toJpeg ? "image/jpeg" : toWebp ? "image/webp" : imageFile.mimeType;
const dataUrl = canvas.toDataURL(outputMime, quality / 100);
```

`toDataURL` takes two arguments: the MIME type and a quality value between 0 and 1. The quality parameter is lossy — higher quality = larger file. For JPEG and WebP it controls the DCT compression level. For PNG it's ignored (PNG is lossless).

## Three Presets + Custom Slider

```typescript
const PRESETS = [
    { id: "extreme",     label: "Extreme",     desc: "Smallest file size",  quality: 30 },
    { id: "recommended", label: "Recommended", desc: "Best balance",        quality: 60 },
    { id: "light",       label: "Light",       desc: "Near-lossless",       quality: 85 },
];
```

Extreme (30%) cuts file size by 70–80% at visible quality loss — fine for thumbnails and previews. Recommended (60%) is the default. Light (85%) is near-imperceptible for most photos. Users can also drag the slider to any value, which sets the preset to "custom".

## Auto-Compress on Upload (Stale Closure Problem)

Images compress automatically when dropped. The catch: `loadFiles` uses a `forEach` with async callbacks inside a `FileReader.onload`. By the time those callbacks fire, React state may have updated — but the closure captures the old values.

The fix is to capture current settings before the loop:

```typescript
const loadFiles = (files: FileList | File[]) => {
    // Capture settings NOW — before any async callbacks fire
    const currentQuality = quality;
    const currentToJpeg  = convertToJpeg;
    const currentToWebp  = convertToWebp;
    const currentResize  = resizeEnabled;
    const currentMaxWidth = maxWidth;

    validFiles.forEach((file) => {
        const reader = new FileReader();
        reader.onload = async (e) => {
            // Uses captured values, not stale closures
            const compressed = await compressSingle(
                newImage, currentQuality, currentToJpeg,
                currentToWebp, currentResize, currentMaxWidth
            );
        };
        reader.readAsDataURL(file);
    });
};
```

This ensures every image compresses with the settings that were active at drop time, not whatever state happens to be set when the async callback fires.

## PNG → JPEG: White Background Fill

PNG supports transparency. JPEG doesn't. If you compress a transparent PNG to JPEG without handling this, the transparent areas become black.

The fix: fill the canvas white before drawing the image:

```typescript
if (toJpeg) {
    ctx.fillStyle = "#FFFFFF";
    ctx.fillRect(0, 0, targetW, targetH);
}
ctx.drawImage(img, 0, 0, targetW, targetH);
```

A PNG compressed to JPEG typically shrinks 60–80% in file size. The tool shows a warning when a PNG is uploaded without WebP or JPEG conversion selected: "PNG — convert to WebP or JPEG for best compression."

## WebP vs JPEG

```typescript
const outputMime = toJpeg ? "image/jpeg" : toWebp ? "image/webp" : imageFile.mimeType;
```

WebP at quality 60 is roughly equivalent in visual quality to JPEG at quality 75, but 25–35% smaller. `canvas.toDataURL("image/webp", 0.6)` handles the encoding — no library needed. The only constraint: old Safari (pre-2020) doesn't support WebP, and iOS 14+ added support. For modern web use, WebP is the better default.

## Resize to Max Width

```typescript
if (resize && mw > 0 && targetW > mw) {
    targetH = Math.round(targetH * (mw / targetW));
    targetW = mw;
}
const canvas = document.createElement("canvas");
canvas.width  = targetW;
canvas.height = targetH;
```

Scaling down before drawing is the biggest possible size reduction — a 4000px wide photo scaled to 1920px loses 77% of its pixels before any quality compression runs.

## Measuring the Compressed Size

`canvas.toDataURL` returns a base64-encoded data URL string. To know the file size, decode the length:

```typescript
function getDataUrlSize(dataUrl: string): number {
    const base64 = dataUrl.split(",")[1];
    const padding = base64.endsWith("==") ? 2 : base64.endsWith("=") ? 1 : 0;
    return Math.floor((base64.length * 3) / 4) - padding;
}
```

Base64 encodes 3 bytes as 4 characters. Padding characters (`=`) at the end mean the last group wasn't a full 3 bytes. This formula gives the exact byte count of the compressed file without needing a Blob or File object.

## Batch Download with JSZip

Single file downloads use `URL.createObjectURL` + anchor click. For multiple files, the tool bundles everything into a ZIP:

```typescript
const downloadAll = async () => {
    const { default: JSZip } = await import("jszip");
    const zip = new JSZip();

    done.forEach((img) => {
        const base64 = img.compressedUrl!.split(",")[1];
        const baseName = img.name.replace(/\.[^.]+$/, "");
        const ext = getOutputExt(img);
        zip.file(`${baseName}-compressed.${ext}`, base64, { base64: true });
    });

    const blob = await zip.generateAsync({ type: "blob" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "compressed-images.zip";
    link.click();
    URL.revokeObjectURL(url);
};
```

`JSZip` is imported dynamically — it only loads when the user actually clicks "Download All". No bundle size cost on initial load.

## Settings Changed State

When a user changes quality after uploading, the new settings don't automatically apply. A `dirtySettings` boolean tracks this:

```typescript
const handleQualityChange = (val: number) => {
    setQuality(val);
    setPreset("custom");
    if (images.length > 0) setDirtySettings(true); // prompt to recompress
};
```

The "Recompress All" button highlights when `dirtySettings` is true, and calls `compressAll()` which runs all images through `compressSingle` again with the current settings using `Promise.all`:

```typescript
const results = await Promise.all(
    images.map((img) => compressSingle(img, quality, convertToJpeg, convertToWebp, resizeEnabled, maxWidth))
);
```

## The Full Picture

Everything runs in the browser. No upload, no server, no file size limits imposed by a backend. The Canvas API handles JPEG and WebP encoding natively. JSZip handles batching. The only network request is the initial page load.

Try it: [Image Compressor → ultimatetools.io](https://ultimatetools.io/tools/image-tools/image-compressor/)