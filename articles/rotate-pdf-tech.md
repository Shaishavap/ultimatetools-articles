# Rotating PDF Pages in the Browser with pdf-lib — Per-Page Rotation, Thumbnail Grid, and Cumulative Angle Tracking

Rotating a PDF page is a metadata change, not a pixel manipulation. PDF pages have a `Rotate` entry in their dictionary — pdf-lib exposes this directly. But building a usable rotation tool involves more than one API call: you need thumbnails, selection state, and correct angle accumulation.

Here's how the [Rotate PDF](https://ultimatetools.io/tools/pdf-tools/rotate-pdf/) tool at Ultimate Tools is built.

## How PDF Rotation Works

A PDF page stores its orientation as a `Rotate` value: 0, 90, 180, or 270 degrees. This is separate from the page's content — the content stream itself doesn't change. Only the display instruction changes.

pdf-lib exposes this cleanly:

```typescript
import { PDFDocument, degrees } from 'pdf-lib';

const rotatePage = (page: PDFPage, angle: 0 | 90 | 180 | 270) => {
  const current = page.getRotation().angle; // 0, 90, 180, or 270
  const newAngle = ((current + angle) % 360) as 0 | 90 | 180 | 270;
  page.setRotation(degrees(newAngle));
};
```

`degrees()` is pdf-lib's wrapper that creates a `Rotation` object. `getRotation().angle` returns the current rotation so you can accumulate it correctly.

## Rendering Thumbnails with PDF.js

Before the user can select pages to rotate, they need to see what the pages look like:

```typescript
import * as pdfjs from 'pdfjs-dist';
pdfjs.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';

const renderThumbnails = async (file: File): Promise<string[]> => {
  const pdf = await pdfjs.getDocument({ data: new Uint8Array(await file.arrayBuffer()) }).promise;
  const thumbs: string[] = [];

  for (let i = 1; i <= pdf.numPages; i++) {
    const page = await pdf.getPage(i);
    const viewport = page.getViewport({ scale: 0.25 });
    const canvas = document.createElement('canvas');
    canvas.width = viewport.width;
    canvas.height = viewport.height;
    await page.render({ canvasContext: canvas.getContext('2d')!, viewport }).promise;
    thumbs.push(canvas.toDataURL('image/jpeg', 0.65));
  }

  return thumbs;
};
```

Scale 0.25 keeps thumbnails small and fast to render. JPEG encoding at 65% quality is fine for thumbnails — users don't need pixel-perfect previews here.

## Selection State: Set of Page Indices

```typescript
const [selected, setSelected] = useState<Set<number>>(new Set());

const togglePage = (i: number) =>
  setSelected(prev => {
    const next = new Set(prev);
    next.has(i) ? next.delete(i) : next.add(i);
    return next;
  });

const selectAll = () =>
  setSelected(new Set(Array.from({ length: numPages }, (_, i) => i)));

const clearSelection = () => setSelected(new Set());
```

The selection is 0-based. The `Set` gives O(1) lookups for highlight rendering.

## Applying Rotation to Selected Pages

The rotation is applied to the pdf-lib document in memory:

```typescript
type RotationAngle = 90 | 180 | 270;

const applyRotation = async (
  file: File,
  pagesToRotate: Set<number>,
  angle: RotationAngle
): Promise<Uint8Array> => {
  const pdfDoc = await PDFDocument.load(new Uint8Array(await file.arrayBuffer()));
  const pages = pdfDoc.getPages();

  for (const index of pagesToRotate) {
    const page = pages[index];
    const current = page.getRotation().angle;
    const next = ((current + angle) % 360) as 0 | 90 | 180 | 270;
    page.setRotation(degrees(next));
  }

  return pdfDoc.save();
};
```

This mutates rotation metadata only — the content streams are untouched, so the output file is nearly the same size as the input.

## Cumulative Angle: Why It Matters

A user might rotate the same page multiple times. Each rotation adds to the current angle, wrapping at 360°:

```
current: 0   + 90 → 90
current: 90  + 90 → 180
current: 180 + 90 → 270
current: 270 + 90 → 0 (wraps)
```

The `% 360` handles this correctly. Without the mod, you'd accumulate values like 360, 450, 540 — which pdf-lib might reject or handle inconsistently.

## Reflecting Rotation in Thumbnails

After the user applies a rotation, the thumbnails should update to show the new orientation. Rather than re-rendering from PDF.js (which would require saving the document first), the tool applies a CSS transform to the thumbnail image:

```typescript
// Track per-page rotation locally in UI state
const [uiRotations, setUiRotations] = useState<number[]>(new Array(numPages).fill(0));

const applyUiRotation = (indices: Set<number>, angle: number) => {
  setUiRotations(prev =>
    prev.map((r, i) => (indices.has(i) ? (r + angle) % 360 : r))
  );
};
```

```tsx
<img
  src={thumbnails[i]}
  style={{ transform: `rotate(${uiRotations[i]}deg)` }}
/>
```

This gives instant visual feedback without re-rendering the thumbnails.

## Downloading the Result

```typescript
const download = (bytes: Uint8Array, name: string) => {
  const blob = new Blob([bytes], { type: 'application/pdf' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = name;
  a.click();
  URL.revokeObjectURL(url);
};
```

## Key Gotchas

| Problem | Solution |
|---|---|
| Rotation accumulates incorrectly | Use `% 360` after each addition |
| PDF already has a rotation | Read with `page.getRotation().angle` first |
| Thumbnails don't reflect rotation | CSS `rotate()` transform on the img element |
| Encrypted PDFs throw on load | Try/catch the `PDFDocument.load` call |
| 0-based vs 1-based page numbers | Internal selection is 0-based; UI labels are 1-based |

PDF rotation is one of the cleanest operations in pdf-lib — it's pure metadata. The complexity is in the UI layer: showing pages, tracking selection, and reflecting changes without re-rendering. The tool is live at [Rotate PDF](https://ultimatetools.io/tools/pdf-tools/rotate-pdf/).