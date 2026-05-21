# Client-Side Only: PDF, Image & Text Tools That Never Touch a Server

There's a pattern worth knowing about: fully client-side processing tools. No upload, no server, no network requests for the actual file processing. The computation runs in your browser using JavaScript, Canvas, WebAssembly, or Web Workers.

This approach has specific advantages:

- **Privacy**: files never leave the device
- **Latency**: no upload/download round trip
- **Cost**: no server infrastructure for file processing
- **Offline capability**: works after the page loads, even without a connection

The trade-offs are real too (browser memory limits, no access to native libraries like Ghostscript), but for a wide class of common operations, client-side is the better choice.

Here's a rundown of tools organised by technology stack, all available at [Ultimate Tools](https://ultimatetools.io/).

---

## PDF Tools — PDF-lib + Web Workers

PDF-lib is a pure JavaScript library for reading and writing PDF files. It's what powers client-side PDF processing without any server dependency.

**[Compress PDF](https://ultimatetools.io/tools/pdf-tools/compress-pdf/)**
Re-encodes embedded images via Canvas API at reduced quality. Runs in a Web Worker to avoid blocking the main thread. Typical reduction: 40–70% for image-heavy PDFs.

**[Merge PDF](https://ultimatetools.io/tools/pdf-tools/merge-pdf/)**
Uses PDF-lib's `copyPages()` to extract pages from multiple source documents and assemble them into a new PDF. Drag-to-reorder with a visual page preview.

**[Split PDF](https://ultimatetools.io/tools/pdf-tools/split-pdf/)**
The inverse of merge — extracts page ranges or individual pages into separate PDF files. Supports "every N pages" splitting for document batching.

**[Remove Pages from PDF](https://ultimatetools.io/tools/pdf-tools/remove-pdf-pages/)**
Loads the PDF, filters out the selected page indices, and saves the result. Straightforward PDF-lib page manipulation.

**[PDF to JPG](https://ultimatetools.io/tools/pdf-tools/pdf-to-jpg/)**
Uses the browser's built-in PDF renderer (via `<canvas>` + PDF.js) to rasterise each page at configurable DPI, then exports as JPEG.

**[PDF to Text](https://ultimatetools.io/tools/pdf-tools/pdf-to-text/)**
Uses PDF.js to extract the text content layer from each page. Returns the text as a string for clipboard copy or .txt download.

**[Protect PDF](https://ultimatetools.io/tools/pdf-tools/protect-pdf/)**
PDF-lib supports AES-256 encryption with owner and user passwords. The encryption key derivation and encryption both happen in browser.

---

## Image Tools — Canvas API

The Canvas API is the browser's 2D drawing surface. It's also the most capable image processing tool available in browser without external dependencies.

**[Image Compressor](https://ultimatetools.io/tools/image-tools/image-compressor/)**
Draws the image onto a Canvas, then exports via `canvas.toBlob({ type: 'image/jpeg', quality })`. Quality parameter (0–1) controls JPEG compression. WebP output uses the same approach with `'image/webp'`.

**[Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/)**
Draws the source image onto a Canvas at target dimensions using `drawImage(img, 0, 0, targetWidth, targetHeight)`. Aspect ratio locking is applied before the draw call.

**[Crop Image](https://ultimatetools.io/tools/image-tools/crop-image/)**
Uses `drawImage(img, sx, sy, sw, sh, 0, 0, dw, dh)` — the full 9-parameter signature that defines both source and destination rectangles.

**[Blur Image](https://ultimatetools.io/tools/image-tools/blur-image/)**
CSS filter or Canvas `filter = 'blur(Npx)'` applied before drawing. Pixelation is achieved by scaling down then scaling up without interpolation.

**[Rotate Image](https://ultimatetools.io/tools/image-tools/rotate-image/)**
Canvas transform: `ctx.translate(cx, cy)`, `ctx.rotate(angle)`, `ctx.translate(-cx, -cy)`. For 90°/180°/270° rotations on PNG/WebP, this is lossless.

**[Image Converter](https://ultimatetools.io/tools/image-tools/image-converter/)**
Loads the source image onto a Canvas, exports to target format via `toBlob()`. PNG→WebP→JPEG conversions are one Canvas call each.

**[Add Watermark](https://ultimatetools.io/tools/image-tools/watermark-image/)**
Draws the source image, then uses `ctx.fillText()` with configurable font, color, opacity, and position for the watermark text layer.

**[HTML to Image](https://ultimatetools.io/tools/image-tools/html-to-image/)**
Uses html-to-image library (SVG foreignObject approach). Clones the target DOM, inlines computed styles, serialises to SVG, renders SVG to Canvas.

---

## Text & Code Tools — Pure JavaScript

These tools have no interesting architecture — they're pure string manipulation in JavaScript. Worth listing because they're often found as SaaS tools despite being trivially implementable client-side.

**[Markdown to HTML](https://ultimatetools.io/tools/coding-tools/markdown-to-html/)** — marked.js  
**[JSON Formatter](https://ultimatetools.io/tools/text-tools/json-formatter/)** — `JSON.parse()` + `JSON.stringify(null, 2)`  
**[JSON to CSV](https://ultimatetools.io/tools/text-tools/json-to-csv/)** — Array key extraction + CSV serialisation  
**[Base64 Encoder/Decoder](https://ultimatetools.io/tools/coding-tools/base64-encoder-decoder/)** — `btoa()` / `atob()` + FileReader for files  
**[URL Encoder/Decoder](https://ultimatetools.io/tools/coding-tools/url-encoder-decoder/)** — `encodeURIComponent()` / `decodeURIComponent()`  
**[Hash Generator](https://ultimatetools.io/tools/coding-tools/hash-generator/)** — Web Crypto API: `crypto.subtle.digest('SHA-256', buffer)`  
**[Case Converter](https://ultimatetools.io/tools/text-tools/case-converter/)** — regex transforms for camelCase, snake_case, Title Case  
**[Code Beautifier](https://ultimatetools.io/tools/coding-tools/code-beautifier/)** — prettier (WASM build) or js-beautify

---

## The Web Crypto API for Hash Generation

One underused browser capability worth highlighting:

```javascript
async function sha256(text) {
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}
```

`crypto.subtle.digest` supports SHA-1, SHA-256, SHA-384, and SHA-512. MD5 is not included (it's cryptographically broken) — MD5 requires a pure-JS implementation. The [Hash Generator](https://ultimatetools.io/tools/coding-tools/hash-generator/) covers MD5 via a JS library and SHA variants via Web Crypto.

---

## OffscreenCanvas for Web Workers

The modern approach for CPU-intensive image processing moves Canvas work to a Web Worker using `OffscreenCanvas`:

```javascript
// Worker
const canvas = new OffscreenCanvas(width, height);
const ctx = canvas.getContext('2d');
ctx.drawImage(imageBitmap, 0, 0);
const blob = await canvas.convertToBlob({ type: 'image/jpeg', quality: 0.7 });
```

This keeps the main thread free and the UI responsive during compression. Browser support: Chrome 69+, Firefox 105+, Safari 16.4+.

---

Client-side processing won't replace every server-side tool — OCR, advanced PDF manipulation, and video processing still benefit from server infrastructure. But for the class of operations described here, the browser is capable and the privacy benefit is real.

Full tool list: [ultimatetools.io](https://ultimatetools.io/)