DEV.TO ARTICLE — Image Compressor (Technical)
================================================================
- dev.to renders markdown natively — paste body as-is
- Set canonical URL in Settings > Canonical URL (leave blank — dev.to is original)
- Cover image: see prompt below
================================================================

TITLE:
Building a Client-Side Image Compressor with Canvas API in Next.js

CANONICAL URL:
(leave blank — dev.to is the original for this article)

TAGS:
javascript, nextjs, webdev, canvas

COVER IMAGE PROMPT:
Generate a dark-themed technical cover image. Dark navy background (#0f172a). Show a visual of the HTML5 Canvas element on the left processing a large image file, with an arrow pointing to a smaller compressed output on the right. Include subtle code snippets: "canvas.toDataURL('image/jpeg', 0.8)" and "JSZip". Style: flat illustration, clean, developer-focused. Wide format (1600x840). No stock photo people.

COVER IMAGE INSTRUCTIONS:
1. Generate using the prompt above in your preferred AI image tool (Midjourney, DALL·E, Ideogram, etc.)
2. Download at 1600x840 or crop to that ratio
3. Upload in dev.to article editor: click the image icon at the top of the editor
4. Set as cover image via "Manage → Cover image" before publishing

PUBLISHED: https://dev.to/shaishav_patel_271fdcd61a/building-a-client-side-image-compressor-with-canvas-api-in-nextjs-5fme

---------- BODY ----------

# Building a Client-Side Image Compressor with Canvas API in Next.js

Most image compression tools work the same way: upload to server, process, download. That means every file leaves the user's device. For a utility tool focused on privacy, I wanted compression to happen entirely in the browser — no server, no upload, no third-party dependencies.

This post covers how I built the [Image Compressor at ultimatetools.io](https://ultimatetools.io/tools/image-tools/image-compressor/) using the browser's native Canvas API, with batch processing support (up to 20 images) and ZIP download via JSZip.

---

## The core idea: Canvas API for compression

The browser's `HTMLCanvasElement.toDataURL()` method is the heart of client-side image compression. Here is the key insight:

```js
canvas.toDataURL('image/jpeg', quality)
```

The second argument — `quality` — is a float between 0 and 1. It controls JPEG compression. At `0.8`, the file is typically 60–80% smaller than the original with no visible quality difference.

This is the same technique used by Squoosh, but we add batch processing on top.

---

## Reading the file

Each image is loaded via `FileReader` or directly from the `File` object using `createObjectURL`:

```ts
const loadImage = (file: File): Promise<HTMLImageElement> => {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve(img);
    img.onerror = reject;
    img.src = URL.createObjectURL(file);
  });
};
```

Using `createObjectURL` is faster than `readAsDataURL` for large files because it doesn't base64-encode the entire file before loading.

---

## Drawing to canvas and exporting

Once the image is loaded, we draw it to an off-screen canvas and export with the target format and quality:

```ts
const compressImage = async (
  file: File,
  quality: number,
  format: 'jpeg' | 'png' | 'original'
): Promise<Blob> => {
  const img = await loadImage(file);
  const canvas = document.createElement('canvas');
  canvas.width = img.naturalWidth;
  canvas.height = img.naturalHeight;

  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(img, 0, 0);

  const mimeType =
    format === 'jpeg' ? 'image/jpeg' :
    format === 'png'  ? 'image/png'  :
    file.type;

  return new Promise((resolve) => {
    canvas.toBlob(
      (blob) => resolve(blob!),
      mimeType,
      mimeType === 'image/jpeg' ? quality : undefined
    );
  });
};
```

A few things worth noting:

**PNG compression**: `toDataURL` and `toBlob` do not accept a quality argument for PNG — PNG is lossless by definition. The canvas API strips unnecessary metadata when re-encoding, which gives a 20–40% reduction on typical screenshots and logos without touching a single pixel.

**WebP input**: The canvas API handles WebP input fine on modern browsers. For output, we currently export JPEG or PNG (WebP output support is browser-dependent and inconsistent across Safari versions).

**Canvas dimensions**: We use `naturalWidth` and `naturalHeight` to preserve the original resolution. No resizing happens — just re-encoding.

---

## Format toggle: JPEG vs PNG vs original

The tool supports three output modes:

```ts
type OutputFormat = 'jpeg' | 'png' | 'original';
```

- `jpeg` — converts to JPEG regardless of input (best for photos)
- `png` — converts to PNG regardless of input (best for logos/screenshots)
- `original` — keeps the input format, just re-encodes at the given quality

For `original` mode, we fall back to the file's MIME type (`file.type`). If the input is WebP and we pass `image/webp` to `toBlob`, Chrome handles it correctly. Safari falls back to PNG silently — acceptable behaviour.

---

## Batch processing: processing images sequentially

With up to 20 images, naively firing all Canvas operations in parallel can cause memory spikes. We process images sequentially to keep memory usage flat:

```ts
const results: CompressedFile[] = [];

for (const file of files) {
  const blob = await compressImage(file, quality, format);
  results.push({
    name: getOutputFilename(file.name, format),
    blob,
    originalSize: file.size,
    compressedSize: blob.size,
  });
}
```

Sequential processing means a slight wait for large batches, but it avoids allocating 20 canvases simultaneously — important on mobile devices with limited heap.

---

## ZIP download with JSZip

For batch download we use [JSZip](https://stuk.github.io/jszip/):

```ts
import JSZip from 'jszip';

const downloadAllAsZip = async (files: CompressedFile[]) => {
  const zip = new JSZip();

  for (const file of files) {
    zip.file(file.name, file.blob);
  }

  const zipBlob = await zip.generateAsync({ type: 'blob' });
  const url = URL.createObjectURL(zipBlob);

  const a = document.createElement('a');
  a.href = url;
  a.download = 'compressed-images.zip';
  a.click();

  URL.revokeObjectURL(url);
};
```

JSZip is pure JavaScript, runs entirely in the browser, and handles 20 files without issue. The ZIP generation happens in the main thread — for very large batches a Web Worker could offload this, but for 20 images it's fast enough.

---

## Calculating compression stats

After compression, we show the reduction percentage for each file:

```ts
const reductionPercent = (
  ((originalSize - compressedSize) / originalSize) * 100
).toFixed(1);
```

We also show the combined stats across all files — total original size vs total compressed size. This is useful feedback for the user and makes the tool feel responsive.

---

## The PNG lossless limitation

One thing worth being explicit about: **PNG compression via canvas is not true lossless optimisation**. Tools like pngquant apply palette quantisation to dramatically reduce PNG file size. The Canvas API cannot do that — it just re-encodes.

For a PNG logo with few colours, pngquant might achieve 70% reduction. Our canvas approach achieves 20–40%. If you need maximum PNG compression, a server-side tool with pngquant is more powerful.

For our use case (privacy-first, no upload), the canvas approach is the right trade-off.

---

## Next.js integration

The component is a standard client component (`'use client'`) using `useRef` for the hidden canvas and `useState` for the file list and settings:

```ts
'use client';

import { useState, useCallback } from 'react';
import { useDropzone } from 'react-dropzone';

export default function ImageCompressor() {
  const [files, setFiles] = useState<CompressedFile[]>([]);
  const [quality, setQuality] = useState(0.8);
  const [format, setFormat] = useState<OutputFormat>('original');

  const onDrop = useCallback(async (acceptedFiles: File[]) => {
    const results = await processFiles(acceptedFiles, quality, format);
    setFiles(results);
  }, [quality, format]);

  const { getRootProps, getInputProps } = useDropzone({
    onDrop,
    accept: { 'image/*': ['.jpg', '.jpeg', '.png', '.webp'] },
    maxFiles: 20,
  });

  // ...render
}
```

No server API routes needed. The entire pipeline — load → canvas → compress → zip — runs in the browser.

---

## Recap

- **Input**: File objects from react-dropzone
- **Processing**: `HTMLCanvasElement.toBlob()` with format + quality control
- **Batch**: Sequential processing to keep memory flat
- **Output**: Individual download or ZIP via JSZip
- **Server**: None required

The live tool is at [Image Compressor](https://ultimatetools.io/tools/image-tools/image-compressor/). Try dropping 10 photos and downloading the ZIP — it's genuinely fast.

If you have questions about the Canvas API behaviour across browsers or the JSZip integration, drop them in the comments.