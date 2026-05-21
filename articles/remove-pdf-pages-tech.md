# Removing PDF Pages in the Browser with pdf-lib — Page Grid UI, Range Input, and Reverse-Index Deletion

Removing pages from a PDF is conceptually simple — keep the pages you want, skip the ones you don't. The implementation has a few non-obvious pieces: rendering thumbnails without a server, building a usable selection UI, and handling page deletion correctly in pdf-lib.

Here's how the [Remove PDF Pages](https://ultimatetools.io/tools/pdf-tools/remove-pdf-pages/) tool at Ultimate Tools works.

## Rendering the Page Grid

The first step is showing the user all pages as thumbnails so they can choose what to remove. This uses PDF.js:

```typescript
import * as pdfjs from 'pdfjs-dist';
pdfjs.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';

const renderThumbnails = async (file: File) => {
  const arrayBuffer = await file.arrayBuffer();
  const pdf = await pdfjs.getDocument({ data: arrayBuffer }).promise;
  const thumbnails: string[] = [];

  for (let i = 1; i <= pdf.numPages; i++) {
    const page = await pdf.getPage(i);
    const viewport = page.getViewport({ scale: 0.3 }); // small for grid display
    const canvas = document.createElement('canvas');
    canvas.width = viewport.width;
    canvas.height = viewport.height;
    await page.render({ canvasContext: canvas.getContext('2d')!, viewport }).promise;
    thumbnails.push(canvas.toDataURL('image/jpeg', 0.7)); // JPEG at 70% for speed
  }

  return thumbnails;
};
```

Scale 0.3 gives small-enough thumbnails for a grid while keeping rendering fast. JPEG encoding at 70% keeps the data URLs compact in memory.

## Selection State

The selection is a `Set<number>` of page indices (0-based):

```typescript
const [selectedPages, setSelectedPages] = useState<Set<number>>(new Set());

const togglePage = (index: number) => {
  setSelectedPages(prev => {
    const next = new Set(prev);
    if (next.has(index)) {
      next.delete(index);
    } else {
      next.add(index);
    }
    return next;
  });
};
```

Using a `Set` makes O(1) membership checks for the thumbnail highlight state:

```tsx
<div
  className={`thumbnail ${selectedPages.has(i) ? 'selected' : ''}`}
  onClick={() => togglePage(i)}
>
  <img src={thumbnails[i]} />
  <span>{i + 1}</span>
</div>
```

## Range Input Parsing

For large PDFs, clicking individual thumbnails is slow. A range input like `1, 5-8, 12` lets users select many pages at once:

```typescript
const parsePageRange = (input: string, totalPages: number): Set<number> => {
  const selected = new Set<number>();
  const parts = input.split(',').map(s => s.trim());

  for (const part of parts) {
    if (part.includes('-')) {
      const [start, end] = part.split('-').map(Number);
      for (let i = start; i <= end && i <= totalPages; i++) {
        selected.add(i - 1); // convert to 0-based
      }
    } else {
      const page = Number(part);
      if (page >= 1 && page <= totalPages) {
        selected.add(page - 1);
      }
    }
  }

  return selected;
};
```

The input is 1-based (user-facing). The internal selection uses 0-based indices to match pdf-lib's API.

## The Deletion: Copy Pages Except the Selected Ones

pdf-lib doesn't have a `removePage` method. The correct approach is to create a new document and copy only the pages you want to keep:

```typescript
const removePages = async (file: File, pagesToRemove: Set<number>): Promise<Uint8Array> => {
  const srcBytes = new Uint8Array(await file.arrayBuffer());
  const srcDoc = await PDFDocument.load(srcBytes);
  const dstDoc = await PDFDocument.create();

  const keepIndices = Array.from(
    { length: srcDoc.getPageCount() },
    (_, i) => i
  ).filter(i => !pagesToRemove.has(i));

  const copiedPages = await dstDoc.copyPagesFrom(srcDoc, keepIndices);
  copiedPages.forEach(page => dstDoc.addPage(page));

  return dstDoc.save();
};
```

`copyPagesFrom` copies content streams, fonts, and resources from the source document. The resulting PDF is self-contained — no references to the original file.

## Why Not Use `removePage()` Directly?

pdf-lib does have `pdfDoc.removePage(index)` — it removes a page from the document's page tree. However, it doesn't remove the underlying page objects and resources from the document, which can leave orphaned objects that inflate the file size.

The copy-to-new-document approach produces a cleaner, smaller output file.

## Preventing the User from Removing All Pages

A guard before deletion:

```typescript
const handleRemove = async () => {
  if (selectedPages.size === 0) return;
  if (selectedPages.size >= totalPages) {
    setError('Cannot remove all pages — at least one page must remain.');
    return;
  }
  // proceed with removal
};
```

## Guarding Against Encrypted PDFs

pdf-lib will throw on password-protected files:

```typescript
try {
  const srcDoc = await PDFDocument.load(srcBytes);
} catch (e) {
  if (String(e).includes('encrypted')) {
    setError('This PDF is password-protected. Remove the password first.');
    return;
  }
  throw e;
}
```

## Key Takeaways

| Problem | Solution |
|---|---|
| Rendering page thumbnails | PDF.js at scale 0.3 |
| Selecting many pages fast | Range input parser (`1, 5-8, 12`) |
| Deleting pages in pdf-lib | Copy-to-new-doc (not `removePage()`) |
| Avoiding file bloat | New document drops orphaned resources |
| Encrypted PDFs | Catch on load, return user-friendly error |

The full tool is live at [Remove PDF Pages](https://ultimatetools.io/tools/pdf-tools/remove-pdf-pages/) — thumbnail grid selection, range input, instant client-side processing.