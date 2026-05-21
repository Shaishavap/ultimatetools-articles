# Converting Images to WebP in the Browser with Canvas API — Quality Control, Format Detection, No Server

WebP images are 25–35% smaller than JPEG and up to 80% smaller than PNG at equivalent visual quality. Browser support is now at 96%+. There's no good reason to still be serving PNG for photos or JPEG where WebP works.

I built a browser-based [Image Converter](https://ultimatetools.io/tools/image-tools/image-converter/) that handles 8 format conversions entirely client-side — including WebP output. No upload, no server, no API key. Here's how the WebP conversion pipeline works.

## The Core: Canvas toBlob()

The entire conversion comes down to one method:

```javascript
canvas.toBlob((blob) => {
  if (!blob) return
  const url = URL.createObjectURL(blob)
  // trigger download
}, 'image/webp', quality)
```

Three parameters:
- **callback** — receives the `Blob` (or `null` if encoding failed)
- **type** — the MIME type of the output format (`'image/webp'`, `'image/jpeg'`, `'image/png'`, etc.)
- **quality** — a float from `0.0` to `1.0`, applies to lossy formats (WebP, JPEG). Ignored for PNG.

The full pipeline: load the image → draw to canvas → call `toBlob()` with the target MIME type.

## Loading the Input Image

```javascript
function loadImage(file: File): Promise<HTMLImageElement> {
  return new Promise((resolve, reject) => {
    const img = new Image()
    const url = URL.createObjectURL(file)
    img.onload = () => {
      URL.revokeObjectURL(url)
      resolve(img)
    }
    img.onerror = reject
    img.src = url
  })
}
```

`URL.createObjectURL()` creates a temporary local URL for the file — no upload, no base64 blob. The image loads from memory. We revoke it in `onload` to free the reference.

## Drawing to Canvas at Natural Resolution

```javascript
async function convertToFormat(
  file: File,
  outputType: string,
  quality: number
): Promise<Blob> {
  const img = await loadImage(file)

  const canvas = document.createElement('canvas')
  canvas.width = img.naturalWidth
  canvas.height = img.naturalHeight

  const ctx = canvas.getContext('2d')!
  ctx.drawImage(img, 0, 0)

  return new Promise((resolve, reject) => {
    canvas.toBlob((blob) => {
      if (!blob) reject(new Error(`Encoding to ${outputType} failed`))
      else resolve(blob)
    }, outputType, quality)
  })
}
```

`naturalWidth` / `naturalHeight` give you the original pixel dimensions regardless of how the image is displayed on screen. This preserves full resolution in the output.

## Handling Transparency: PNG and WebP vs JPEG

WebP supports transparency (alpha channel). JPEG does not.

When converting a PNG with transparency to JPEG, you need to fill the transparent areas — otherwise they render as black:

```javascript
const ctx = canvas.getContext('2d')!

if (outputType === 'image/jpeg') {
  // Fill transparent background with white before drawing
  ctx.fillStyle = '#ffffff'
  ctx.fillRect(0, 0, canvas.width, canvas.height)
}

ctx.drawImage(img, 0, 0)
```

When converting to WebP, no fill is needed — WebP preserves the alpha channel natively. This makes PNG → WebP lossless-ish conversions particularly useful: you get the transparency support of PNG with dramatically smaller file sizes.

## Quality Slider: Mapping 1–100 to the toBlob Range

`toBlob()` expects quality as `0.0–1.0`. A UI slider typically runs `1–100`. The mapping:

```javascript
const blobQuality = sliderValue / 100
```

For WebP, quality `0.8` (80%) is a good default — visually indistinguishable from the original for most photos, with 60–70% file size reduction.

```javascript
// Quality comparison for a typical 4MB photo:
// quality: 1.0  → ~1.8MB WebP  (still 55% smaller than original PNG)
// quality: 0.8  → ~380KB WebP  (90% smaller, no visible difference)
// quality: 0.5  → ~180KB WebP  (visible compression artifacts)
// quality: 0.1  → ~60KB WebP   (heavily degraded)
```

The sweet spot for most use cases is `0.75–0.85`.

## Detecting WebP Encoding Support

Encoding to WebP (not just decoding) requires browser support. Chrome has had it since 2012. Firefox added it in v96 (2022). Safari added it in v14 (2020).

To detect if the browser can encode WebP:

```javascript
function supportsWebPEncoding(): boolean {
  const canvas = document.createElement('canvas')
  canvas.width = 1
  canvas.height = 1
  return canvas.toDataURL('image/webp').startsWith('data:image/webp')
}
```

If the browser doesn't support WebP encoding, `toDataURL('image/webp')` falls back to PNG and returns `data:image/png`. This 1×1 canvas test catches it without any async work.

In practice at 96%+ global support, this check is mostly a safety net. But it's worth having to avoid silently producing PNG output when WebP was requested.

## Getting the File Size Before and After

One useful UX feature: showing the original vs converted file size so users understand the benefit.

```javascript
const originalSize = file.size // bytes

const blob = await convertToFormat(file, 'image/webp', 0.8)
const convertedSize = blob.size

const saving = Math.round((1 - convertedSize / originalSize) * 100)
console.log(`${saving}% smaller`) // e.g. "73% smaller"
```

`File` extends `Blob`, so both have a `.size` property in bytes. No async work needed to get the original size.

## Triggering the Download

```javascript
function downloadBlob(blob: Blob, originalName: string, ext: string) {
  const baseName = originalName.replace(/\.[^.]+$/, '')
  const filename = `${baseName}.${ext}`

  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()
  URL.revokeObjectURL(url)
}
```

Strip the original extension, append the new one. The `a.download` attribute tells the browser to download rather than navigate. Revoke the object URL immediately after clicking — the download is already queued.

## Format Support Matrix

| Input → Output | Transparency | Lossy | Notes |
|---|---|---|---|
| PNG → WebP | ✅ preserved | optional | Best size/quality tradeoff |
| JPEG → WebP | n/a | ✅ | ~30% smaller than JPEG |
| PNG → JPEG | ❌ white fill | ✅ | Fill background before draw |
| JPEG → PNG | n/a | ❌ | Lossless but larger |
| * → PNG | ✅ preserved | ❌ | Quality param ignored |

## What This Doesn't Cover

The Canvas API approach has one limitation: it re-encodes. If you convert a JPEG to WebP, you're decoding the JPEG (with its existing compression artifacts) and re-encoding to WebP. You don't get back the original quality — you get a WebP of a JPEG.

For the best WebP output, convert from the highest-quality source available (original RAW export, lossless PNG, or high-quality JPEG).

For true lossless WebP encoding (not just transparency support), the Canvas API isn't sufficient — you'd need WebAssembly-compiled libwebp. For most web use cases (compressing photos for upload, converting icons, optimizing blog images), the Canvas approach is entirely adequate.

## Full Pipeline

```javascript
async function handleConversion(
  file: File,
  outputFormat: 'webp' | 'jpeg' | 'png',
  quality: number
) {
  const mimeMap = {
    webp: 'image/webp',
    jpeg: 'image/jpeg',
    png: 'image/png',
  }

  const mimeType = mimeMap[outputFormat]

  if (outputFormat === 'webp' && !supportsWebPEncoding()) {
    throw new Error('WebP encoding not supported in this browser')
  }

  const blob = await convertToFormat(file, mimeType, quality / 100)
  downloadBlob(blob, file.name, outputFormat)

  return {
    originalSize: file.size,
    convertedSize: blob.size,
    saving: Math.round((1 - blob.size / file.size) * 100),
  }
}
```

No dependencies. No server. Works offline after the page loads.

---

Try it: [Image Converter → ultimatetools.io](https://ultimatetools.io/tools/image-tools/image-converter/)

*Part of [Ultimate Tools](https://ultimatetools.io/) — free, privacy-first browser tools.*