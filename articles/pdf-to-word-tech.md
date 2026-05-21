# Building a PDF to Word Converter in Next.js with pdfjs-dist and docx

We needed a PDF to Word converter for [Ultimate Tools](https://ultimatetools.io/). Most browser-based approaches hit a wall: `pdfjs-dist` runs fine in the browser for rendering, but extracting structured text and assembling a real `.docx` file on the client is messy — DOCX generation libraries aren't designed for the browser, and the output quality varies. The cleaner path is a Next.js API route.

Here's how we built it: pdfjs-dist for text extraction on the server, the `docx` library for DOCX assembly, and a Node.js polyfill trick to make pdfjs happy outside the browser.

---

## The Stack

- **pdfjs-dist** (legacy build) — Mozilla's PDF renderer, used here for text extraction only
- **docx** — generates `.docx` files from structured JavaScript objects
- **Next.js API route** — Node.js runtime, handles multipart form upload

---

## The Problem: pdfjs in Node.js Wants Browser APIs

pdfjs-dist is primarily built for browsers. Its module initialisation references browser-only globals — `DOMMatrix`, `Path2D`, and `ImageData` — even when you only want text extraction (no rendering, no canvas). In Node.js these don't exist, so the import crashes before you even call anything.

The fix is to stub them on `globalThis` before importing pdfjs:

```ts
if (typeof (globalThis as Record<string, unknown>).DOMMatrix === 'undefined') {
    (globalThis as Record<string, unknown>).DOMMatrix = class DOMMatrix {
        is2D = true; isIdentity = true;
        transformPoint(p: unknown) { return p; }
    };
}
if (typeof (globalThis as Record<string, unknown>).Path2D === 'undefined') {
    (globalThis as Record<string, unknown>).Path2D = class Path2D {};
}
if (typeof (globalThis as Record<string, unknown>).ImageData === 'undefined') {
    (globalThis as Record<string, unknown>).ImageData = class ImageData {
        constructor(public width: number, public height: number) {}
    };
}
```

These stubs satisfy the module initialisation check. Text extraction never actually uses `DOMMatrix` math or canvas operations at runtime — those paths only activate during page rendering.

---

## Loading pdfjs with a Local Worker

pdfjs needs a worker script. In the browser you point it at a CDN URL. In Node.js, `file://` URLs work inside dynamic `import()`:

```ts
const pdfjsLib = require('pdfjs-dist/legacy/build/pdf.mjs');

const workerPath = path.join(
    process.cwd(),
    'node_modules/pdfjs-dist/legacy/build/pdf.worker.mjs'
).replace(/\\/g, '/');

pdfjsLib.GlobalWorkerOptions.workerSrc = `file:///${workerPath}`;
```

We use the **legacy build** (`pdfjs-dist/legacy/build/pdf.mjs`) — this is the recommended path for Node.js. The standard build assumes a modern browser environment more aggressively.

---

## Extracting Text: Y-Coordinate Line Grouping

pdfjs returns text as a flat array of items, each with a position encoded in a transform matrix. The Y coordinate is `item.transform[5]`.

The challenge: items on the same visual line don't share an exact Y value — floating point means they might differ by 1–2 units. We round to the nearest integer and group by that:

```ts
const items = textContent.items as Array<{ str: string; transform: number[] }>;
const lineMap = new Map<number, string[]>();

for (const item of items) {
    if (!item.str) continue;
    const y = Math.round(item.transform[5]);
    const bucket = lineMap.get(y) ?? [];
    bucket.push(item.str);
    lineMap.set(y, bucket);
}

// PDF Y=0 is at the bottom — sort descending to read top-to-bottom
const sortedKeys = Array.from(lineMap.keys()).sort((a, b) => b - a);
```

Each group of items at the same rounded Y becomes one line of text. Sorting keys in descending order maps the PDF coordinate system (origin at bottom-left) to reading order (top to bottom).

---

## Assembling the DOCX

The `docx` library builds Word documents from a tree of objects — `Document`, `Section`, `Paragraph`, `TextRun`, etc. We map each extracted line to a `Paragraph`:

```ts
import { Document, Packer, Paragraph, TextRun, HeadingLevel, PageBreak, AlignmentType } from 'docx';

const docChildren: Paragraph[] = [];

// Title from filename
docChildren.push(
    new Paragraph({
        text: baseName,
        heading: HeadingLevel.HEADING_1,
        alignment: AlignmentType.CENTER,
        spacing: { after: 300 },
    })
);

for (let pageNum = 1; pageNum <= numPages; pageNum++) {
    const page = await pdfDoc.getPage(pageNum);
    const textContent = await page.getTextContent();

    if (pageNum > 1) {
        docChildren.push(new Paragraph({ children: [new PageBreak()] }));
        docChildren.push(
            new Paragraph({
                text: `— Page ${pageNum} —`,
                heading: HeadingLevel.HEADING_2,
                alignment: AlignmentType.CENTER,
            })
        );
    }

    // ... Y-coordinate grouping from above ...

    for (const key of sortedKeys) {
        const lineText = (lineMap.get(key) ?? []).join(' ').trim();
        if (!lineText) continue;

        docChildren.push(
            new Paragraph({
                children: [new TextRun({ text: lineText, size: 22 })],
                spacing: { after: 80 },
            })
        );
    }
}

const doc = new Document({
    creator: 'UltimateTools.io',
    title: baseName,
    sections: [{ properties: {}, children: docChildren }],
});

const docxBuffer = await Packer.toBuffer(doc);
```

`Packer.toBuffer()` returns a `Buffer` — we send it directly as the HTTP response with the correct DOCX MIME type.

---

## The API Route

```ts
export const runtime = 'nodejs';
const MAX_SIZE = 20 * 1024 * 1024; // 20 MB

export async function POST(req: NextRequest) {
    const formData = await req.formData();
    const file = formData.get('file') as File | null;

    if (!file) return NextResponse.json({ error: 'No file uploaded.' }, { status: 400 });
    if (file.type !== 'application/pdf') return NextResponse.json({ error: 'Only PDF files are supported.' }, { status: 400 });
    if (file.size > MAX_SIZE) return NextResponse.json({ error: 'File too large. Max 20MB.' }, { status: 413 });

    const arrayBuffer = await file.arrayBuffer();

    const pdfDoc = await pdfjsLib.getDocument({
        data: new Uint8Array(arrayBuffer),
        useSystemFonts: true,
        disableFontFace: true,
    }).promise;

    // ... text extraction + DOCX assembly ...

    return new NextResponse(new Uint8Array(docxBuffer), {
        status: 200,
        headers: {
            'Content-Type': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            'Content-Disposition': `attachment; filename="${outputName}"`,
            'Content-Length': String(docxBuffer.byteLength),
        },
    });
}
```

Two pdfjs options worth noting:
- `useSystemFonts: true` — avoids font loading errors in Node.js (no font face parsing needed for text extraction)
- `disableFontFace: true` — same reason; skips font rendering setup entirely

---

## What Works Well, What Doesn't

**Works well:**
- Text-heavy PDFs: contracts, reports, documentation
- Preserves paragraph-level structure and page breaks
- Heading detection from filename (not from PDF metadata)

**Limitations:**
- Multi-column layouts linearize to single-column (Y grouping doesn't account for X position across columns)
- Tables extract as sequential text rows, not structured table elements
- Embedded images are not included — text extraction only
- Scanned PDFs (image-only) return empty output — no OCR

For most real-world use cases — editing a contract, repurposing a report, updating a document — the output is clean and immediately editable.

---

## Result

The full pipeline: multipart upload → pdfjs text extraction → Y-coordinate line grouping → docx paragraph assembly → DOCX download. No external APIs, no third-party conversion services, no per-request costs.

Try it: [PDF to Word](https://ultimatetools.io/tools/pdf-tools/pdf-to-word/)

The full source for the API route is part of [Ultimate Tools](https://ultimatetools.io/) — a free browser toolkit for PDFs, images, QR codes, and developer utilities.