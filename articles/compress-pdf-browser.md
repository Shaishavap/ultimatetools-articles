# Compress PDF in the Browser Without a Server — How It Works (PDF-lib + Web Workers)

Most PDF processing tools send your file to a server. The browser makes a multipart POST request, the server runs LibreOffice or Ghostscript or a Python library, and you download the result.

There's a better approach for many PDF operations: process the file entirely in the browser using PDF-lib and Web Workers. No server involved. No file transmission. Works offline once the page is loaded.

This post explains how browser-side PDF compression works and when it's the right choice.

---

## The Architecture

Client-side PDF processing in the browser uses three components:

1. **[PDF-lib](https://pdf-lib.js.org/)** — a pure JavaScript PDF manipulation library. Reads and writes PDF structure without any native dependencies.
2. **Web Workers** — moves the CPU-intensive processing off the main thread so the UI stays responsive.
3. **Canvas API** — used to re-encode embedded images at reduced quality.

The processing flow:

```
User drops PDF
    → FileReader reads it as ArrayBuffer
    → ArrayBuffer passed to Web Worker via postMessage()
    → Worker uses PDF-lib to parse the PDF
    → Worker extracts embedded image data
    → Worker re-encodes images via Canvas API at lower quality
    → Worker re-embeds compressed images into PDF-lib document
    → Worker returns the new ArrayBuffer
    → Main thread creates a Blob URL for download
```

---

## What "Compression" Actually Means for PDFs

PDFs don't have a single compression knob. What most PDF compressors actually do:

**Image re-encoding**: The biggest wins come from re-encoding embedded JPEG images at lower quality. A JPEG at quality 90 is roughly 3× larger than the same image at quality 60 with minimal perceptible difference for most document content.

**Content stream compression**: PDF-lib applies zlib compression to content streams. Most PDFs already have this applied, so gains here are minimal.

**Font subsetting**: Embedding only the glyphs actually used from a font, rather than the full font file. Significant savings for PDFs using large CJK fonts.

**Object deduplication**: Some PDF generators embed the same resource multiple times. Deduplication finds and merges these.

For typical office documents, image re-encoding is where 80-90% of the compression gain comes from.

---

## Code: PDF Image Compression with PDF-lib

```javascript
import { PDFDocument, decodePDFRawStream } from 'pdf-lib';

async function compressPdf(inputArrayBuffer, quality = 0.7) {
  const pdfDoc = await PDFDocument.load(inputArrayBuffer);
  
  const pages = pdfDoc.getPages();
  
  for (const page of pages) {
    const { node } = page;
    
    // Get all XObjects (images) on this page
    const xObjects = node.XObject?.();
    if (!xObjects) continue;
    
    for (const [name, xObjectRef] of Object.entries(xObjects.dict)) {
      const xObject = pdfDoc.context.lookup(xObjectRef);
      
      if (xObject?.Subtype?.name !== 'Image') continue;
      
      // Get image dimensions
      const width = xObject.Width?.numberValue();
      const height = xObject.Height?.numberValue();
      
      if (!width || !height) continue;
      
      // Decode image data
      const rawData = decodePDFRawStream(xObject).decode();
      
      // Re-encode via Canvas at target quality
      const canvas = new OffscreenCanvas(width, height);
      const ctx = canvas.getContext('2d');
      
      const imageData = new ImageData(
        new Uint8ClampedArray(rawData), width, height
      );
      ctx.putImageData(imageData, 0, 0);
      
      const blob = await canvas.convertToBlob({
        type: 'image/jpeg',
        quality
      });
      const compressedData = new Uint8Array(await blob.arrayBuffer());
      
      // Replace image bytes in PDF
      xObject.contents = compressedData;
      xObject.dict.set(pdfDoc.context.obj('Filter'), pdfDoc.context.obj('DCTDecode'));
    }
  }
  
  return await pdfDoc.save();
}
```

Note: The above is a simplified illustration. Real-world PDF image extraction requires handling various filter types (DCTDecode for JPEG, FlateDecode for PNG-embedded images, JPXDecode for JPEG 2000), colour space conversions, and different stream encoding combinations. The [Compress PDF](https://ultimatetools.io/tools/pdf-tools/compress-pdf/) tool at Ultimate Tools handles these edge cases.

---

## Web Worker Integration

Running PDF-lib on the main thread freezes the UI for large PDFs. Move it to a Worker:

```javascript
// compress-worker.js
import { PDFDocument } from 'pdf-lib';

self.onmessage = async ({ data: { buffer, quality } }) => {
  try {
    const result = await compressPdf(buffer, quality);
    self.postMessage({ success: true, buffer: result }, [result.buffer]);
  } catch (err) {
    self.postMessage({ success: false, error: err.message });
  }
};

// Main thread
const worker = new Worker('/compress-worker.js', { type: 'module' });
worker.postMessage({ buffer: pdfArrayBuffer, quality: 0.6 }, [pdfArrayBuffer]);
worker.onmessage = ({ data }) => {
  if (data.success) {
    const blob = new Blob([data.buffer], { type: 'application/pdf' });
    const url = URL.createObjectURL(blob);
    // Trigger download
  }
};
```

The `[pdfArrayBuffer]` in the `postMessage` call is the transferables list — it transfers ownership of the buffer to the Worker without copying, which matters for large PDFs.

---

## When to Use Server-Side vs Client-Side

| Factor | Client-Side | Server-Side |
|---|---|---|
| File privacy | Better — file never leaves device | Depends on service |
| Large files (>100MB) | Limited by browser memory | No limit |
| Scanned PDFs (image-heavy) | Works well | Works well |
| Font subsetting | Limited in PDF-lib | Full support (Ghostscript) |
| Compression ratio | Good (60-80% for image PDFs) | Better (80-90% possible) |
| Infrastructure cost | Zero | Server required |
| Works offline | Yes (after page load) | No |
| GDPR/privacy compliance | Easier | More complex |

For documents under ~50MB with embedded images, browser-side compression typically achieves 40–70% reduction — sufficient for most use cases.

---

## Try It

The [Compress PDF](https://ultimatetools.io/tools/pdf-tools/compress-pdf/) tool at Ultimate Tools implements this approach — PDF-lib + Web Worker + Canvas re-encoding. Free, no upload, works in your browser.

Source patterns worth studying: the [PDF-lib documentation](https://pdf-lib.js.org/) and the [PDF-lib GitHub examples](https://github.com/Hopding/pdf-lib/tree/master/apps/node/examples) cover the full API including encryption, form filling, and page manipulation.

Other browser-based PDF tools using the same architecture: [Merge PDF](https://ultimatetools.io/tools/pdf-tools/merge-pdf/), [Split PDF](https://ultimatetools.io/tools/pdf-tools/split-pdf/), [Remove Pages from PDF](https://ultimatetools.io/tools/pdf-tools/remove-pdf-pages/).