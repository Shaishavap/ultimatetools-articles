# How We Run All PDF Operations in the Browser — pdf-lib, No Server, No Upload

Every PDF tool at Ultimate Tools — merge, split, rotate, compress, reorder, watermark, protect — runs entirely in the browser. No file ever leaves your device. Here's how client-side PDF processing works with [pdf-lib](https://pdf-lib.js.org/) and what the architecture looks like.

---

## Why Client-Side PDF Processing

**Privacy:** Financial documents, legal contracts, medical records. Users don't want these on a stranger's server. Client-side processing is the only honest answer to "where does my file go?"

**Speed:** Network upload of a 30MB PDF takes time. Local processing is instant — no round-trip.

**Cost:** Server-side PDF processing at scale (CPU, memory, storage) is expensive. Shifting it to the client eliminates that cost entirely.

**The tradeoff:** Client-side is limited by the user's device RAM and CPU. A 500-page PDF with embedded high-res images can exhaust memory on low-end devices. Our tools cap file input at 100MB and show a warning for operations that will be slow.

---

## The Core Library: pdf-lib

[pdf-lib](https://pdf-lib.js.org/) is a pure JavaScript PDF library that runs in both Node.js and the browser. It handles:
- Loading and parsing existing PDFs
- Modifying page order, rotation, metadata
- Merging multiple PDFs
- Splitting into separate documents
- Embedding fonts, images, and form fields

It doesn't handle rendering PDF pages to images (that requires pdf.js) or advanced compression (that requires native binaries). But for structural operations — merge, split, rotate, reorder, basic watermark — it covers everything.

---

## Loading a PDF

```typescript
import { PDFDocument } from 'pdf-lib';

async function loadPdf(file: File): Promise<PDFDocument> {
  const buffer = await file.arrayBuffer();
  return PDFDocument.load(buffer, {
    ignoreEncryption: false, // throw on password-protected PDFs
  });
}
```

`file.arrayBuffer()` reads the entire file into memory. For a 50MB PDF, this allocates 50MB in the browser's heap. The parsed `PDFDocument` object is typically another 2–3× the file size in memory. This is why large file limits matter for client-side processing.

---

## Merge: Copying Pages Between Documents

```typescript
async function mergePdfs(files: File[]): Promise<Uint8Array> {
  const merged = await PDFDocument.create();

  for (const file of files) {
    const doc = await loadPdf(file);
    const pages = await merged.copyPages(doc, doc.getPageIndices());
    pages.forEach(page => merged.addPage(page));
  }

  return merged.save();
}
```

`copyPages` copies page objects from one document into another, including all embedded fonts, images, and resources referenced by those pages. `doc.getPageIndices()` returns `[0, 1, 2, ...]` for all pages.

The result is a `Uint8Array` — the raw bytes of the merged PDF. Pass it to a `Blob` for download.

---

## Split: Extracting Page Subsets

```typescript
async function extractPages(file: File, pageNumbers: number[]): Promise<Uint8Array> {
  const source = await loadPdf(file);
  const output = await PDFDocument.create();

  // Convert 1-based user input to 0-based indices
  const indices = pageNumbers.map(n => n - 1).filter(i => i >= 0 && i < source.getPageCount());

  const pages = await output.copyPages(source, indices);
  pages.forEach(page => output.addPage(page));

  return output.save();
}
```

To split a PDF into individual pages:

```typescript
async function splitIntoPages(file: File): Promise<Uint8Array[]> {
  const source = await loadPdf(file);
  const results: Uint8Array[] = [];

  for (let i = 0; i < source.getPageCount(); i++) {
    const singlePage = await PDFDocument.create();
    const [page] = await singlePage.copyPages(source, [i]);
    singlePage.addPage(page);
    results.push(await singlePage.save());
  }

  return results;
}
```

Each extracted PDF is a new document containing only the copied page and its resources.

---

## Reorder Pages

```typescript
async function reorderPages(file: File, newOrder: number[]): Promise<Uint8Array> {
  // newOrder is 1-based: [3, 1, 2] = move page 3 first, then 1, then 2
  const source = await loadPdf(file);
  const output = await PDFDocument.create();

  const indices = newOrder.map(n => n - 1);
  const pages   = await output.copyPages(source, indices);
  pages.forEach(page => output.addPage(page));

  return output.save();
}
```

`copyPages` accepts an array of indices in any order — the pages are returned in the order specified. This makes reordering as simple as passing the desired sequence.

---

## Memory Management

For large multi-file merges, process and discard documents one at a time:

```typescript
async function mergeInChunks(files: File[]): Promise<Uint8Array> {
  const merged = await PDFDocument.create();

  for (const file of files) {
    // Process one file at a time — let previous go out of scope
    const doc   = await PDFDocument.load(await file.arrayBuffer());
    const pages = await merged.copyPages(doc, doc.getPageIndices());
    pages.forEach(p => merged.addPage(p));
    // doc goes out of scope here — eligible for GC
  }

  return merged.save();
}
```

JavaScript's garbage collector will reclaim memory from unreferenced `PDFDocument` objects, but GC timing is non-deterministic. For very large batches, consider processing in groups of 5–10 files and monitoring `performance.memory.usedJSHeapSize` to abort if memory gets critically high.

---

## Downloading the Result

```typescript
function downloadPdf(bytes: Uint8Array, filename: string): void {
  const blob = new Blob([bytes], { type: 'application/pdf' });
  const url  = URL.createObjectURL(blob);

  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();

  URL.revokeObjectURL(url);
}
```

The PDF tools at Ultimate Tools use this same pattern for [Merge PDF](https://ultimatetools.io/tools/pdf-tools/merge-pdf/), [Split PDF](https://ultimatetools.io/tools/pdf-tools/split-pdf/), [Rotate PDF](https://ultimatetools.io/tools/pdf-tools/rotate-pdf/), [Remove PDF Pages](https://ultimatetools.io/tools/pdf-tools/remove-pdf-pages/), and the full [PDF Studio](https://ultimatetools.io/tools/pdf-tools/pdf-studio/). All operations run entirely in the browser — your PDFs never leave your device.