# How We Moved PDF Processing from Server to Browser (and Killed OOM Crashes)

This is a real story from today. No theory — just a debugging session that went from a mysterious `403`, to a silent `500`, to discovering our server was dying from memory exhaustion on every PDF export.

---

## The Setup

We run [Ultimate Tools](https://ultimatetools.io/) — a collection of 24 free, privacy-first browser-based utilities. One of our tools is a **PDF Studio**: organize pages, add watermarks, apply digital signatures (eSign), set metadata, compress, and split documents.

The export flow was simple:

1. User uploads a PDF → saved to server temp directory
2. User applies operations (signatures, reordering, watermarks)
3. User clicks **Download** → `POST /api/pdf/apply/` → server processes with `pdf-lib` → returns the PDF

Worked perfectly in local dev. On production (Hostinger Node.js hosting), it completely broke.

---

## Step 1: The 403 That Made No Sense

The first error was a `403` with a 60ms response time.

```
POST https://ultimatetools.io/api/pdf/apply/ [HTTP/2 403 60ms]
```

Response headers:

```
server: hcdn
content-type: text/plain
content-length: 10
```

60ms. That's not our app responding — our Next.js server takes at least 100ms to cold-start. Something upstream was blocking the request before it ever reached Node.js.

**The culprit:** Hostinger's CDN (`hcdn`) silently blocks large `application/json` POST bodies. Our payload was ~40 KB of JSON containing base64-encoded signature images. The CDN rejected it without forwarding to the app.

**The fix:** Switch to `multipart/form-data`. CDNs treat file uploads differently.

Client side:

```javascript
// Before — blocked by CDN
const res = await fetch('/api/pdf/apply/', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload)
});

// After — passes through fine
const fd = new FormData();
fd.append('data', JSON.stringify(payload));
const res = await fetch('/api/pdf/apply/', {
  method: 'POST',
  body: fd
});
```

Server side (Next.js App Router):

```typescript
export async function POST(req: NextRequest) {
  const contentType = req.headers.get('content-type') || '';
  let body: Record<string, unknown>;

  if (contentType.includes('multipart/form-data')) {
    const formData = await req.formData();
    body = JSON.parse(formData.get('data') as string);
  } else {
    body = await req.json();
  }
}
```

403 gone. Now we got a 500.

---

## Step 2: The 500 With Zero Logs

The new error:

```
server: hcdn
content-type: text/html
retry-after: 60
status: 500
```

Two things stood out:

- `content-type: text/html` — Next.js returns `application/json` for API errors. This HTML came from Hostinger's infrastructure, not our app.
- `retry-after: 60` — the server telling us to back off for 60 seconds.

And the runtime logs? **Nothing.** No `console.error`. No stack trace. Complete silence.

We added verbose logging to the catch block — still nothing. The process was crashing **before the catch block could execute**.

---

## Step 3: Finding the Real Cause

We checked the temp directory on the server. The uploaded file was there:

```
351abec9-6170-45a5-b95f-9cd1abf71f15.pdf    7.34 MiB
414b82b2-6bd1-4f37-9132-d3edc212f8fe.pdf    7.34 MiB
```

**7.34 MB scanned PDF.** That's the moment it clicked.

Here's what `pdf-lib` does when processing a PDF server-side:

```
1. fs.readFile()           →  7.34 MB  (raw bytes in memory)
2. PDFDocument.load()      → ~35 MB   (pdf-lib internal object model)
3. PDFDocument.create()    → ~35 MB   (new document being built)
4. copyPages()             → ~35 MB   (copied page structures)
5. embedPng() per sig      → ~10 MB   (decoded image data)
6. newDoc.save()           → ~15 MB   (Uint8Array output)
                             --------
                             ~137 MB  total
```

Hostinger's Node.js process memory limit is tight. The OS killed the process instantly — no SIGTERM, no graceful shutdown, no logs. Hostinger's Passenger process manager returned a generic 500 HTML page with `retry-after: 60`.

> **Silent OOM. No stack trace. No warning.**

---

## The Real Fix: Move It to the Browser

The browser has no enforced memory limit on JavaScript. A 7 MB PDF processed with `pdf-lib` in V8 is completely fine — and the user's machine does the work, not our server.

**Before:**

```
Upload PDF → server temp file → [server processes with pdf-lib] → download
```

**After:**

```
Upload PDF → server temp file + store ArrayBuffer client-side
Export    → [browser processes with pdf-lib] → direct download
```

### Step 1: Store raw bytes in Zustand on upload

```typescript
if (file.type === 'application/pdf') {
  const arrayBuffer = await file.arrayBuffer();
  setRawPdfBytes(docId, arrayBuffer.slice(0));
  newPages = await loadPdfAndGenerateThumbnails(arrayBuffer, docId);
}
```

### Step 2: Export entirely in the browser

```typescript
const rawDocBytes = rawPdfBytes[firstDocId];

if ((type === 'pdf' || type === 'compress') && rawDocBytes) {
  const { PDFDocument, degrees, rgb } = await import('pdf-lib');

  const srcDoc = await PDFDocument.load(rawDocBytes);
  const outDoc = await PDFDocument.create();

  const copied = await outDoc.copyPages(srcDoc, targetPages.map(p => p.originalIndex));
  for (let i = 0; i < copied.length; i++) {
    copied[i].setRotation(degrees(targetPages[i].rotation));
    outDoc.addPage(copied[i]);
  }

  for (const sig of signatures) {
    const imgBytes = Uint8Array.from(
      atob(sig.imageBase64.split(',')[1]),
      c => c.charCodeAt(0)
    );
    const img = await outDoc.embedPng(imgBytes);
    outDoc.getPages()[sig.pageIndex].drawImage(img, {
      x: sig.x, y: sig.y,
      width: sig.width, height: sig.height,
      rotate: degrees(sig.rotation),
      opacity: sig.opacity
    });
  }

  const finalBytes = await outDoc.save({ useObjectStreams: true });

  const url = URL.createObjectURL(
    new Blob([finalBytes], { type: 'application/pdf' })
  );
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  a.remove();
  URL.revokeObjectURL(url);

  return;
}
```

### Step 3: Clean up server files on session end

```typescript
useEffect(() => {
  const sendCleanup = () => {
    const ids = Object.values(documents).map(d => d.originalFileId);
    if (ids.length === 0) return;
    navigator.sendBeacon(
      '/api/pdf/cleanup/',
      new Blob([JSON.stringify({ fileIds: ids })], { type: 'application/json' })
    );
  };

  window.addEventListener('beforeunload', sendCleanup);
  return () => {
    window.removeEventListener('beforeunload', sendCleanup);
    sendCleanup();
  };
}, []);
```

---

## Results

| | Before | After |
|---|---|---|
| 7 MB PDF export | Server OOM → 500 | ~1 second in browser |
| Server memory | Spikes to limit | Flat |
| Logs on failure | Silent crash | N/A — no server call |
| Works offline | No | Yes |

---

## Key Takeaways

**1. CDN-level blocks are silent and fast.**
A 60ms `403` with `server: hcdn` and a 10-byte body means the CDN rejected it — your app never saw it. Switch to `multipart/form-data` for large payloads.

**2. OOM crashes leave no logs.**
If you see a `500` with `retry-after` header, `text/html` content type, and zero `console.error` output — the process was killed by the OS before it could log anything.

**3. The browser is a powerful compute environment.**
`pdf-lib`, `ffmpeg.wasm`, `pdf.js` — these all run in the browser. If your server is doing heavy transformation on user files, ask whether the user's machine could do it instead. No round trip, no memory pressure, no cost.

**4. `navigator.sendBeacon` for cleanup on page close.**
Regular `fetch` in `beforeunload` is unreliable — browsers cancel in-flight requests on unload. `sendBeacon` is guaranteed to complete.

---

If you want to try the tool that caused all this debugging, it's live at [Ultimate Tools — Free eSign PDF](https://ultimatetools.io/tools/pdf-tools/esign-pdf/) — free, no account required.

*Have you hit silent OOM crashes in production? How did you diagnose them? Drop it in the comments.*