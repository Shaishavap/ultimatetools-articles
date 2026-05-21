# Building an Image Format Converter in the Browser — Canvas API, MIME Type Detection, and Lossless PNG Handling

Browser-based image format conversion uses the Canvas API as a universal decode-then-re-encode pipeline. The interesting problems are: what formats the browser can natively decode, how to handle the PNG alpha channel when converting to JPEG, and how to give users control over output quality.

Here's how the [Image Converter](https://ultimatetools.io/tools/image-tools/image-converter/) at Ultimate Tools is built.

## The Core Pipeline

```typescript
const convertImage = async (
  file: File,
  outputFormat: 'image/jpeg' | 'image/png' | 'image/webp' | 'image/gif',
  quality: number // 0–1, only applies to JPEG and WebP
): Promise<Blob> => {
  const bitmap = await createImageBitmap(file);

  const canvas = document.createElement('canvas');
  canvas.width = bitmap.width;
  canvas.height = bitmap.height;

  const ctx = canvas.getContext('2d')!;

  // Fill white background before drawing if converting to JPEG
  if (outputFormat === 'image/jpeg') {
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
  }

  ctx.drawImage(bitmap, 0, 0);

  return new Promise((resolve, reject) => {
    canvas.toBlob(
      blob => blob ? resolve(blob) : reject(new Error('Conversion failed')),
      outputFormat,
      outputFormat === 'image/png' ? undefined : quality
    );
  });
};
```

`createImageBitmap` is the decode step — it handles JPEG, PNG, WebP, GIF (first frame), BMP, AVIF, and TIFF in browsers that support them. `canvas.toBlob` is the encode step.

## The Alpha Channel Problem: PNG → JPEG

JPEG has no alpha channel. If you draw a transparent PNG onto a canvas and export as JPEG, the transparent areas become black:

```
canvas default background = transparent (rgba(0,0,0,0))
JPEG sees transparent = rgb(0,0,0) = black
```

The fix is filling the canvas with white before drawing:

```typescript
if (outputFormat === 'image/jpeg') {
  ctx.fillStyle = '#ffffff';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
}
ctx.drawImage(bitmap, 0, 0);
```

This gives the expected result: transparent areas appear white in the JPEG, as if the image had been placed on a white background.

## MIME Type Detection

The input format is detected from `file.type`, but file.type can be wrong (user renamed the file). The tool validates by checking magic bytes as a fallback:

```typescript
const detectFormat = async (file: File): Promise<string> => {
  // Trust MIME type if set
  if (file.type && file.type !== 'application/octet-stream') {
    return file.type;
  }

  // Read first 12 bytes for magic number check
  const buffer = await file.slice(0, 12).arrayBuffer();
  const bytes = new Uint8Array(buffer);

  // PNG: 89 50 4E 47
  if (bytes[0] === 0x89 && bytes[1] === 0x50) return 'image/png';
  // JPEG: FF D8 FF
  if (bytes[0] === 0xFF && bytes[1] === 0xD8) return 'image/jpeg';
  // WebP: 52 49 46 46 ... 57 45 42 50
  if (bytes[0] === 0x52 && bytes[1] === 0x49 && bytes[8] === 0x57) return 'image/webp';
  // GIF: 47 49 46 38
  if (bytes[0] === 0x47 && bytes[1] === 0x49) return 'image/gif';

  return file.type || 'unknown';
};
```

## Quality Control

Quality only applies to lossy formats. PNG is always lossless — passing a quality value to `canvas.toBlob` for PNG is ignored by browsers anyway, but being explicit prevents confusion:

```typescript
const getQualityParam = (format: string, quality: number): number | undefined => {
  if (format === 'image/jpeg' || format === 'image/webp') {
    return quality; // 0–1
  }
  return undefined; // PNG ignores quality
};
```

WebP supports both lossy and lossless modes via quality:
- `quality = 1.0` → lossless WebP (larger file, no compression artifacts)
- `quality < 1.0` → lossy WebP (smaller, quality-dependent compression)

## GIF Output Limitation

`canvas.toBlob('image/gif')` is not supported by any major browser. GIF encoding requires a JavaScript library (such as `gif.js` or `omggif`).

The tool handles this by using a pure-JS GIF encoder as a fallback:

```typescript
if (outputFormat === 'image/gif') {
  return encodeStaticGif(canvas); // library-based encoding
}
```

For the common case of converting GIF to another format (not to GIF), the input GIF is decoded by `createImageBitmap`, which extracts the first frame. This is a static export — animations are not preserved.

## Showing Input Metadata

Before conversion, the tool displays the input image's format, dimensions, and file size:

```typescript
const getImageMetadata = async (file: File) => {
  const bitmap = await createImageBitmap(file);
  return {
    format: file.type,
    width: bitmap.width,
    height: bitmap.height,
    sizeBytes: file.size,
  };
};
```

Showing both input and output sizes lets users see the compression gain from converting to a more efficient format.

## Downloading the Result

```typescript
const downloadBlob = (blob: Blob, originalName: string, outputFormat: string) => {
  const ext = outputFormat.split('/')[1]; // 'image/jpeg' → 'jpeg'
  const name = originalName.replace(/\.[^.]+$/, '') + '.' + (ext === 'jpeg' ? 'jpg' : ext);
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = name;
  a.click();
  URL.revokeObjectURL(url);
};
```

The filename extension is updated to match the output format.

## Key Gotchas

| Problem | Solution |
|---|---|
| Transparent PNG → JPEG shows black | Fill canvas white before drawing |
| file.type unreliable | Validate with magic byte check |
| GIF encoding not in Canvas API | Use a JS library for GIF output |
| WebP quality = 1.0 is lossless | Document this for the quality slider |
| Large images OOM | Use `createImageBitmap` (off-main-thread decoding) |

The full tool is live at [Image Converter](https://ultimatetools.io/tools/image-tools/image-converter/) — drop any image, pick your output format, adjust quality, download.