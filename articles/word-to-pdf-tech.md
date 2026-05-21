# Building a Word to PDF Converter in Next.js — Server-Side DOCX Text Extraction with Mammoth and PDF Generation with PDFKit

Converting a .docx file to PDF server-side comes down to two steps: extract the text content from the DOCX format, then build a new PDF from that text. The [Word to PDF](https://ultimatetools.io/tools/pdf-tools/word-to-pdf/) converter uses mammoth.js for extraction and PDFKit for generation — both running in a Next.js API route.

## Why Server-Side?

DOCX is a ZIP archive containing XML files. Parsing it client-side is possible but adds ~500KB to the bundle (mammoth is not small). Server-side keeps the bundle clean, and the conversion runs in Node.js where Buffer and stream APIs are natively available — exactly what PDFKit needs.

## The Route Setup

```typescript
import mammoth from 'mammoth';
import PDFDocument from 'pdfkit';

export const runtime = 'nodejs'; // PDFKit requires Node.js streams
const MAX_SIZE = 20 * 1024 * 1024;
```

`export const runtime = 'nodejs'` is required. PDFKit uses Node.js streams internally — it cannot run in the Edge Runtime.

Both `mammoth` and `pdfkit` are declared as `serverExternalPackages` in `next.config.mjs` to prevent webpack from bundling them:

```js
serverExternalPackages: ['pdfkit', 'mammoth'],
```

Both use `__dirname` internally to locate font and asset files. If webpack bundles them, those path resolutions break.

## Receiving the File

```typescript
const formData = await req.formData();
const file = formData.get('file') as File | null;

const isDocx = file.name.toLowerCase().endsWith('.docx') ||
    DOCX_MIME_TYPES.includes(file.type);
```

Browsers sometimes send `.docx` files as `application/octet-stream` depending on the OS. The validation checks both MIME type and file extension as a fallback.

## Step 1: Extract Text with Mammoth

```typescript
const arrayBuffer = await file.arrayBuffer();
const buffer = Buffer.from(arrayBuffer);

const mammothResult = await mammoth.extractRawText({ buffer });
const rawText = mammothResult.value ?? '';
```

`mammoth.extractRawText()` strips all formatting and returns the plain text content of the document. Mammoth also has `convertToHtml()` which preserves bold, italic, headings, and lists — but that requires parsing HTML to build the PDF, which adds significant complexity.

For a general-purpose converter, raw text is the right tradeoff: reliable output across all DOCX structures, no edge cases from unusual formatting.

## Step 2: Paragraph Detection

DOCX paragraphs are separated by double newlines in mammoth's raw text output:

```typescript
const paragraphs = rawText
    .split(/\n{2,}/)
    .map(p => p.replace(/\n/g, ' ').trim())
    .filter(Boolean);
```

The `.replace(/\n/g, ' ')` collapses single newlines within a paragraph (soft line breaks in Word) into spaces. Double newlines become paragraph breaks. Empty strings are filtered out.

## Step 3: Build the PDF with PDFKit

PDFKit uses a streaming API — you write content to a `PDFDocument` object and collect the output chunks:

```typescript
const pdfBuffer = await new Promise<Buffer>((resolve, reject) => {
    const doc = new PDFDocument({
        margin: 72,   // 1 inch in PDF points
        size: 'A4',
        info: {
            Title: baseName,
            Author: 'UltimateTools.io',
            Creator: 'UltimateTools.io Word to PDF Converter',
        },
    });

    const chunks: Buffer[] = [];
    doc.on('data', (chunk: Buffer) => chunks.push(chunk));
    doc.on('end', () => resolve(Buffer.concat(chunks)));
    doc.on('error', reject);

    // Title
    doc.fontSize(18)
        .font('Helvetica-Bold')
        .text(baseName, { align: 'center' });
    doc.moveDown(1.5);

    // Body paragraphs
    doc.fontSize(11).font('Helvetica');

    for (const para of paragraphs) {
        doc.text(para, {
            align: 'justify',
            lineGap: 2,
        });
        doc.moveDown(0.75);
    }

    doc.end();
});
```

Key PDFKit concepts:

**Streaming collection** — `doc.on('data')` fires as PDFKit writes chunks. `doc.on('end')` fires when done. `Buffer.concat(chunks)` assembles the final PDF bytes.

**Points, not pixels** — PDFKit uses PDF points (1/72 of an inch). `margin: 72` = 1 inch margin. `fontSize(11)` = 11pt body text.

**Built-in fonts** — `'Helvetica'` and `'Helvetica-Bold'` are PDF standard fonts, no file loading needed. For custom fonts, PDFKit supports `.ttf` and `.otf` via `doc.registerFont()`.

**`doc.end()`** — must be called to flush the final chunk and trigger the `'end'` event.

## Sending the PDF Response

```typescript
return new NextResponse(new Uint8Array(pdfBuffer), {
    status: 200,
    headers: {
        'Content-Type': 'application/pdf',
        'Content-Disposition': `attachment; filename="${outputName}"`,
        'Content-Length': String(pdfBuffer.byteLength),
    },
});
```

`Content-Disposition: attachment` triggers a file download in the browser rather than inline display. The client receives a binary blob.

## Client Side: Blob Download

```typescript
const response = await fetch('/api/convert/word-to-pdf/', {
    method: 'POST',
    body: form, // FormData with the .docx file
});

const blob = await response.blob();
const url = URL.createObjectURL(blob);
setPdfUrl(url);
```

The anchor tag uses the object URL with a `download` attribute:

```tsx
<a href={pdfUrl} download={pdfFileName}>Download PDF</a>
```

Object URLs are cleaned up on reset via `URL.revokeObjectURL(prevUrlRef.current)` — a ref tracks the previous URL to prevent memory leaks across multiple conversions.

## What Mammoth Doesn't Preserve

`extractRawText` drops everything except text content: no bold, no italic, no headings hierarchy, no tables, no images, no custom fonts. For a general converter this is fine — you get a clean, readable PDF every time regardless of how the original was styled.

If you need formatting fidelity, `mammoth.convertToHtml()` preserves semantic structure (headings become `<h1>–<h6>`, bold becomes `<strong>`). You'd then parse the HTML and map elements to PDFKit method calls — doable but significantly more complex.

## Deployment Note

On Hostinger (and similar Node.js hosts with memory constraints), PDFKit's approach of buffering the entire PDF in memory is fine for typical documents. The risk is very large files with embedded images — but this converter handles text-only content, so memory usage stays low regardless of document length.

Try it: [Word to PDF → ultimatetools.io](https://ultimatetools.io/tools/pdf-tools/word-to-pdf/)