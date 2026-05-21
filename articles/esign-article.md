How to Sign PDFs in the Browser Without Uploading Anything


========== TAGS (max 4) ==========

javascript, webdev, privacy, productivity


========== BODY (paste in dev.to body editor — raw markdown) ==========

Most free PDF signing tools have the same problem: they upload your document to a third-party server, process it there, and hand it back. For a random form that's fine. For a client contract or an NDA, it's not.

Here's how we built an eSign tool where the PDF never leaves the browser — and what we learned doing it.

---

## The Core Idea

The entire signing flow runs client-side using [pdf-lib](https://pdf-lib.js.org/):

1. User opens a PDF → read as `ArrayBuffer` in the browser
2. User draws a signature on a `<canvas>` element
3. Signature canvas is exported as a PNG data URL
4. `pdf-lib` embeds the PNG onto the correct page at the correct coordinates
5. Output PDF is downloaded via `URL.createObjectURL` — no server involved

```typescript
const { PDFDocument, degrees } = await import('pdf-lib');

const srcDoc = await PDFDocument.load(rawPdfBytes);
const outDoc = await PDFDocument.create();

// Copy pages preserving rotation
const copied = await outDoc.copyPages(srcDoc, pageIndices);
copied.forEach(page => outDoc.addPage(page));

// Embed each signature
for (const sig of signatures) {
  const imgBytes = Uint8Array.from(
    atob(sig.imageBase64.split(',')[1]),
    c => c.charCodeAt(0)
  );
  const img = await outDoc.embedPng(imgBytes);
  outDoc.getPages()[sig.pageIndex].drawImage(img, {
    x: sig.x,
    y: sig.y,
    width: sig.width,
    height: sig.height,
    rotate: degrees(sig.rotation),
    opacity: sig.opacity,
  });
}

const finalBytes = await outDoc.save({ useObjectStreams: true });

// Trigger download
const url = URL.createObjectURL(
  new Blob([finalBytes], { type: 'application/pdf' })
);
const a = document.createElement('a');
a.href = url;
a.download = 'signed.pdf';
a.click();
URL.revokeObjectURL(url);
```

Zero server calls. Zero uploads. The signed PDF is generated entirely in V8.

---

## Storing the PDF Bytes

One gotcha: `pdf-lib` needs the raw bytes of the original PDF. After the user picks a file, store the `ArrayBuffer` in state before passing it to the thumbnail renderer — both need it and you don't want to read the file twice.

```typescript
const arrayBuffer = await file.arrayBuffer();

// Store raw bytes for later export
setRawPdfBytes(docId, arrayBuffer.slice(0)); // .slice(0) clones the buffer

// Generate page thumbnails from the same bytes
await loadPdfAndGenerateThumbnails(arrayBuffer, docId);
```

We use Zustand for this. The raw bytes stay in memory for the session, which is fine — the user already has the file open locally.

---

## The Signature Canvas

The signature pad is a `<canvas>` element that captures pointer events. On export, `canvas.toDataURL('image/png')` gives you the base64 PNG to embed.

A few things worth handling:

- **Multi-touch**: prevent page scroll while drawing on mobile (`touch-action: none` on the canvas)
- **Pressure sensitivity**: `pointerEvent.pressure` gives you line width variation on supported devices
- **Clear vs undo**: store each stroke as a separate path array so you can undo stroke by stroke rather than clearing everything

---

## Why Not Server-Side?

We originally processed PDFs server-side. It broke in production due to OOM crashes on large files — `pdf-lib` needs roughly 5x the file size in memory to process a PDF (loading, copying, embedding, saving all at once).

On a constrained Node.js host (Hostinger, in our case), a 7 MB scanned PDF pushed the process past the memory limit. The OS killed it instantly — no error log, no stack trace, just a `500` with `retry-after: 60` from the infrastructure layer.

Moving to client-side eliminated the problem entirely. The user's browser handles the memory pressure on their own machine. A 50 MB PDF processes in under two seconds.

> If your server is transforming files that the user already has locally, ask yourself whether the browser could do it instead.

---

## Cleanup

The original PDF file gets saved to a server temp directory during upload (needed for DOCX-to-PDF conversion and Word/Text export). We clean it up when the user leaves using `navigator.sendBeacon` — the only fetch API guaranteed to complete on page unload:

```typescript
useEffect(() => {
  const cleanup = () => {
    navigator.sendBeacon(
      '/api/pdf/cleanup/',
      new Blob([JSON.stringify({ fileIds })], { type: 'application/json' })
    );
  };
  window.addEventListener('beforeunload', cleanup);
  return () => {
    window.removeEventListener('beforeunload', cleanup);
    cleanup(); // also fires on in-app navigation
  };
}, [fileIds]);
```

Regular `fetch` in `beforeunload` gets cancelled by the browser before it completes. `sendBeacon` does not.

---

## Try It

The tool is live at [eSign PDF](https://ultimatetools.io/tools/pdf-tools/esign-pdf/) — free, no account, no upload.

If you're building something similar and hit edge cases with `pdf-lib` (font embedding, form fields, encrypted PDFs), drop a comment — happy to share what we ran into.