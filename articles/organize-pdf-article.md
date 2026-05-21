DEV.TO ARTICLE — PDF Page Organizer (Technical)
================================================================
- Paste body using Ctrl + Shift + V (plain text) in dev.to editor
- Set canonical URL in Settings > Canonical URL
- Cover image: PDF thumbnails grid, drag-to-reorder theme
================================================================

TITLE:
Building a Browser-Based PDF Page Organizer with pdf-lib and pdfjs-dist

CANONICAL URL:
(leave blank — dev.to is the original for this article)

TAGS:
javascript, nextjs, webdev, pdf

COVER IMAGE PROMPT:
Generate a dark-themed technical cover image. Show a grid of PDF page thumbnails on a dark navy background (#0f172a). Some thumbnails have a rotation arrow overlay, one has a red trash/delete icon, and a few are highlighted with an indigo selection ring. A subtle drag handle (grip dots) is visible on one card. Style: flat illustration, clean, minimal. No text in the image. Wide format (1600x840).

COVER IMAGE INSTRUCTIONS:
1. Generate using the prompt above in your preferred AI image tool (Midjourney, DALL·E, Ideogram, etc.)
2. Download at 1600x840 or crop to that ratio
3. Upload in dev.to article editor: click the image icon at the top of the editor
4. Set as cover image via "Manage → Cover image" before publishing

PUBLISHED: https://dev.to/shaishav_patel_271fdcd61a/building-a-browser-based-pdf-page-organizer-with-pdf-lib-and-pdfjs-dist-422k

---------- BODY ----------

# Building a Browser-Based PDF Page Organizer with pdf-lib and pdfjs-dist

Rotate a page, remove a blank one, extract a few pages into a separate file — these are everyday PDF tasks that most tools send to a server. In this post I'll walk through how I built a fully client-side PDF page organizer in Next.js, covering three operations: **rotate**, **remove**, and **extract pages** — all without uploading the file anywhere.

The live tool is at [Remove PDF Pages](https://ultimatetools.io/tools/pdf-tools/remove-pdf-pages/) and [Rotate PDF](https://ultimatetools.io/tools/pdf-tools/rotate-pdf/).

---

## The architecture

Three libraries do the work:

- **pdfjs-dist** — renders each page to a `<canvas>` for thumbnail preview
- **pdf-lib** — manipulates and exports the final PDF
- **@tanstack/react-virtual** — virtualizes the thumbnail grid so 100+ page PDFs don't tank performance
- **Zustand** — manages all page state with undo/redo

The flow: load file → generate thumbnails → user organizes pages → export with pdf-lib.

---

## Loading the PDF and generating thumbnails

Everything starts with `file.arrayBuffer()`. The raw bytes are stored for the export step, and pdfjs renders thumbnails from the same bytes:

```ts
const arrayBuffer = await file.arrayBuffer();

// Keep raw bytes — pdf-lib needs these at export time
setRawPdfBytes(docId, arrayBuffer.slice(0));

// Generate thumbnails with pdfjs-dist
const pages = await loadPdfAndGenerateThumbnails(arrayBuffer, docId);
addDocument({ id: docId, filename: file.name }, pages);
```

The thumbnail generator renders each page at 1.5x scale to a canvas and immediately converts to a JPEG data URL:

```ts
const loadPdfAndGenerateThumbnails = async (
  dataBuffer: ArrayBuffer,
  docId: string
) => {
  const pdf = await pdfjs.getDocument({ data: new Uint8Array(dataBuffer) }).promise;
  const newPages: Page[] = [];

  for (let i = 1; i <= pdf.numPages; i++) {
    const page = await pdf.getPage(i);
    const viewport = page.getViewport({ scale: 1.5 });

    const canvas = document.createElement('canvas');
    canvas.width = viewport.width;
    canvas.height = viewport.height;

    const ctx = canvas.getContext('2d')!;
    await page.render({ canvasContext: ctx, viewport }).promise;

    const thumbnailUrl = canvas.toDataURL('image/jpeg', 0.6);

    // Free GPU memory immediately — critical for multi-page docs
    canvas.width = 0;
    canvas.height = 0;

    newPages.push({
      id: `page-${docId}-${i}`,
      docId,
      originalIndex: i - 1, // 0-based — what pdf-lib's copyPages() expects
      rotation: 0,
      thumbnailUrl,
      width: viewport.width,
      height: viewport.height,
    });
  }

  return newPages;
};
```

Two things worth noting:
- `originalIndex` is stored **0-based** — that's what pdf-lib's `copyPages()` expects
- The canvas is explicitly zeroed out after each page (`canvas.width = 0`). Without this, the GPU holds the texture for every page simultaneously. On a 50-page PDF this causes OOM crashes on mobile

---

## The state model

All page state lives in a Zustand store with a normalized structure:

```ts
interface Page {
  id: string;
  docId: string;
  originalIndex: number; // 0-based index in the source PDF
  rotation: number;      // 0, 90, 180, 270
  thumbnailUrl?: string;
  width: number;
  height: number;
}

interface PdfStudioState {
  pages: Record<string, Page>;
  rawPdfBytes: Record<string, ArrayBuffer>;
  pageSequence: string[];  // ordered array of page IDs — single source of truth
  selection: string[];     // selected page IDs
}
```

`pageSequence` is the key. It's an ordered array of page IDs. Every operation — rotate, remove, reorder — works by producing a new `pageSequence`. The `pages` map is just a lookup table.

---

## Operation 1: Rotate

Rotation is just a state update on the page's `rotation` field. The thumbnail displays the CSS rotation immediately, and pdf-lib applies it on export.

```ts
rotatePage: (pageId) => set((state) => {
  const page = state.pages[pageId];
  if (!page) return state;

  return {
    pages: {
      ...state.pages,
      [pageId]: { ...page, rotation: (page.rotation + 90) % 360 }
    }
  };
}),
```

In the thumbnail, rotation is applied as a CSS transform — no re-render needed:

```tsx
<img
  src={page.thumbnailUrl}
  alt={`Page ${index + 1}`}
  style={{
    transform: `rotate(${page.rotation}deg)`,
    transition: 'transform 0.3s ease'
  }}
/>
```

At export time, pdf-lib applies the actual rotation to the PDF page object:

```ts
const [copied] = await outDoc.copyPages(srcDoc, [page.originalIndex]);
copied.setRotation(pdfDegrees(page.rotation));
outDoc.addPage(copied);
```

---

## Operation 2: Remove pages

Remove just filters the page out of `pageSequence` and deletes it from the `pages` map:

```ts
removePage: (pageId) => set((state) => {
  const newPages = { ...state.pages };
  delete newPages[pageId];

  return {
    pages: newPages,
    pageSequence: state.pageSequence.filter(id => id !== pageId),
    selection: state.selection.filter(id => id !== pageId),
  };
}),
```

Bulk remove (select multiple + delete) just calls this in a loop:

```ts
const handleDeleteSelected = () => {
  selection.forEach(id => removePage(id));
  clearSelection();
};
```

Because `pageSequence` drives the grid render, the thumbnails disappear instantly without any additional state.

---

## Operation 3: Extract pages

Extract is the same as a normal export, but only operates on the selected pages instead of the full `pageSequence`:

```ts
const triggerExport = async (isExtract = false) => {
  const { PDFDocument, degrees: pdfDegrees } = await import('pdf-lib');

  const srcDoc = await PDFDocument.load(rawPdfBytes[docId]);
  const outDoc = await PDFDocument.create();

  // Extract: use selection. Normal save: use full pageSequence.
  const targetPageIds = isExtract && selection.length > 0
    ? selection
    : pageSequence;

  const targetPages = targetPageIds.map(id => ({
    originalIndex: pages[id].originalIndex,
    rotation: pages[id].rotation,
  }));

  const copied = await outDoc.copyPages(
    srcDoc,
    targetPages.map(p => p.originalIndex)
  );

  for (let i = 0; i < copied.length; i++) {
    copied[i].setRotation(pdfDegrees(targetPages[i].rotation));
    outDoc.addPage(copied[i]);
  }

  const bytes = await outDoc.save();
  const url = URL.createObjectURL(
    new Blob([bytes], { type: 'application/pdf' })
  );

  const a = document.createElement('a');
  a.href = url;
  a.download = isExtract ? 'extracted-pages.pdf' : 'document.pdf';
  document.body.appendChild(a);
  a.click();
  a.remove();
  URL.revokeObjectURL(url);
};
```

The extract button only appears in the toolbar when `selection.length > 0`, so the user can select any subset of pages and pull them out as a new document.

---

## Drag-to-reorder

The thumbnail grid supports drag-to-reorder. The reorder logic is a simple splice on `pageSequence`:

```ts
reorderPages: (fromIndex, toIndex) => set((state) => {
  const newSequence = [...state.pageSequence];
  const [moved] = newSequence.splice(fromIndex, 1);
  newSequence.splice(toIndex, 0, moved);
  return { pageSequence: newSequence };
}),
```

Multi-select drag (holding Ctrl/Cmd to select multiple pages, then dragging) uses a different function that preserves relative order within the selection:

```ts
reorderPagesGroup: (draggedPageId, targetPageId, selectedIds) => set((state) => {
  const sequence = [...state.pageSequence];
  const selectedSet = new Set(selectedIds);

  const selectedInOrder = sequence.filter(id => selectedSet.has(id));
  const remaining = sequence.filter(id => !selectedSet.has(id));

  const draggedOrigIdx = sequence.indexOf(draggedPageId);
  const targetOrigIdx = sequence.indexOf(targetPageId);
  const targetInRemaining = remaining.indexOf(targetPageId);

  // Insert after target if dragging forward, before if dragging backward
  let insertIdx = targetInRemaining === -1
    ? remaining.length
    : draggedOrigIdx < targetOrigIdx
    ? targetInRemaining + 1
    : targetInRemaining;

  return {
    pageSequence: [
      ...remaining.slice(0, insertIdx),
      ...selectedInOrder,
      ...remaining.slice(insertIdx),
    ]
  };
}),
```

---

## Virtualizing the thumbnail grid

For PDFs with 50-100+ pages, rendering all thumbnails at once causes layout jank. The grid uses `@tanstack/react-virtual` to only render visible rows:

```ts
const virtualizer = useVirtualizer({
  count: rowCount,          // total rows based on columns and page count
  getScrollElement: () => parentRef.current,
  estimateSize: () => itemHeight + gap,
  overscan: 2,              // render 2 extra rows above/below viewport
});
```

Column count is calculated from container width with a `ResizeObserver`:

```ts
useEffect(() => {
  const calculateColumns = () => {
    if (!parentRef.current) return;
    const usableWidth = parentRef.current.clientWidth - 64;
    const cols = Math.floor((usableWidth + gap) / (itemWidth + gap));
    setColumns(Math.max(1, cols));
  };

  calculateColumns();
  const observer = new ResizeObserver(calculateColumns);
  if (parentRef.current) observer.observe(parentRef.current);
  return () => observer.disconnect();
}, [zoom, itemWidth]);
```

This recalculates on both resize and zoom level changes, so the grid reflows correctly when the user zooms in or out.

---

## Undo/redo

Every mutating action (rotate, remove, reorder) pushes a snapshot to a `past` stack before applying:

```ts
// Inside any mutating action:
const pastState = {
  pages: { ...state.pages },
  pageSequence: [...state.pageSequence],
};

return {
  ...newState,
  past: [...state.past, pastState],
  future: [], // clear redo stack on new action
};
```

Undo pops from `past`, pushes current state to `future`:

```ts
undo: () => set((state) => {
  if (state.past.length === 0) return state;

  const newPast = [...state.past];
  const previousState = newPast.pop()!;

  return {
    ...previousState,
    past: newPast,
    future: [
      { pages: state.pages, pageSequence: state.pageSequence },
      ...state.future,
    ],
  };
}),
```

---

## Gotchas

**1. `originalIndex` stays fixed — never update it**

`originalIndex` points to the page's position in the *original* source PDF. It never changes when you reorder, rotate, or remove pages in the UI. Only `pageSequence` changes. When exporting, you map `pageSequence → originalIndex` to tell pdf-lib which pages to copy. If you accidentally update `originalIndex` on reorder, the export produces wrong pages.

**2. Don't re-render thumbnails on rotation**

Rotation only changes the CSS `transform` on the `<img>`. You do NOT need to re-render the pdfjs thumbnail. The thumbnail image is always the unrotated version; the visual rotation is purely CSS. pdf-lib applies the actual rotation on export via `setRotation(degrees(n))`. This is faster and avoids re-running the canvas pipeline.

**3. pdfjs worker in Next.js App Router**

pdfjs needs a worker. Without it, rendering blocks the main thread. In App Router, the worker must be loaded client-side only:

```ts
// In a 'use client' file or useEffect
import * as pdfjs from 'pdfjs-dist';
pdfjs.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.js';
```

Copy `node_modules/pdfjs-dist/build/pdf.worker.min.js` to your `public/` folder, or reference a CDN URL.

**4. Selection state survives reorders — fix it**

When pages are reordered, their IDs don't change, so `selection` stays valid automatically. But when a page is *removed*, you must filter it out of `selection` too:

```ts
removePage: (pageId) => set((state) => ({
  pages: newPages,
  pageSequence: state.pageSequence.filter(id => id !== pageId),
  selection: state.selection.filter(id => id !== pageId), // ← don't forget this
})),
```

---

## Result

All three operations — rotate, remove, extract — run entirely in the browser. The PDF bytes never leave the device. Rendering is fast thanks to pdfjs thumbnails + virtual scrolling. Undo/redo covers every operation.

Try it: [Remove PDF Pages](https://ultimatetools.io/tools/pdf-tools/remove-pdf-pages/)

Free, no account required.