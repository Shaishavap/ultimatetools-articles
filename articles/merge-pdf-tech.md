# Merging PDFs in the Browser with pdf-lib — copyPagesFrom, File Order State, and Memory Considerations

PDF merging sounds simple — concatenate documents. The implementation has a few non-obvious pieces: how pdf-lib's page copying actually works, how to manage ordered file state in React, and what happens to memory when you load several large PDFs at once.

Here's how the [Merge PDF](https://ultimatetools.io/tools/pdf-tools/merge-pdf/) tool at Ultimate Tools is built.

## The Core API: copyPagesFrom

pdf-lib doesn't have a "merge" function. The correct approach is to create a new document and copy pages from each source document into it:

```typescript
import { PDFDocument } from 'pdf-lib';

const mergePdfs = async (files: File[]): Promise<Uint8Array> => {
  const mergedDoc = await PDFDocument.create();

  for (const file of files) {
    const bytes = new Uint8Array(await file.arrayBuffer());
    const srcDoc = await PDFDocument.load(bytes);

    const pageIndices = srcDoc.getPageIndices(); // [0, 1, 2, ...]
    const copiedPages = await mergedDoc.copyPagesFrom(srcDoc, pageIndices);

    for (const page of copiedPages) {
      mergedDoc.addPage(page);
    }
  }

  return mergedDoc.save();
};
```

`copyPagesFrom` copies the page content streams, resources (fonts, images, color spaces), and annotations from the source document into the destination document. The result is self-contained — no references back to the source files.

## Why Not Just Add Pages Directly?

You might expect something like `mergedDoc.addPage(srcDoc.getPage(0))` to work. It doesn't. Pages from one PDFDocument can't be added to another directly — they contain internal references (fonts, resource dictionaries) that are scoped to their original document. Trying to cross-add pages without copying throws an error or produces a broken PDF.

`copyPagesFrom` is the only correct path. It rewires all internal references to point to the destination document's resource dictionary.

## File Order State in React

The user needs to control page order before merging. The tool maintains an ordered array of file entries:

```typescript
type PdfEntry = {
  id: string;       // stable identity for React keys
  file: File;
  name: string;
  pageCount: number;
  sizeKb: number;
};

const [entries, setEntries] = useState<PdfEntry[]>([]);

const addFiles = async (newFiles: File[]) => {
  const newEntries: PdfEntry[] = await Promise.all(
    newFiles.map(async file => {
      const bytes = new Uint8Array(await file.arrayBuffer());
      const doc = await PDFDocument.load(bytes);
      return {
        id: crypto.randomUUID(),
        file,
        name: file.name,
        pageCount: doc.getPageCount(),
        sizeKb: Math.round(file.size / 1024),
      };
    })
  );
  setEntries(prev => [...prev, ...newEntries]);
};
```

Loading each PDF to get its page count costs memory upfront, but lets you show page counts in the file list — helpful when the user is assembling a multi-file merge.

## Drag-to-Reorder

The file list supports drag-and-drop reordering using HTML5 drag events:

```typescript
const dragIndexRef = useRef<number | null>(null);

const onDragStart = (index: number) => {
  dragIndexRef.current = index;
};

const onDrop = (targetIndex: number) => {
  const from = dragIndexRef.current;
  if (from === null || from === targetIndex) return;

  setEntries(prev => {
    const next = [...prev];
    const [moved] = next.splice(from, 1);
    next.splice(targetIndex, 0, moved);
    return next;
  });

  dragIndexRef.current = null;
};
```

The `id` field on each entry ensures React's reconciler doesn't confuse elements when the array order changes — without a stable key, React can reassign DOM state to the wrong list item.

## Memory Considerations

Each PDF is loaded as a `Uint8Array` in memory. For large files, this adds up fast — five 20MB PDFs means 100MB+ in browser memory just for the source bytes, plus overhead for the parsed document objects.

The approach to keep memory manageable:

```typescript
const mergePdfsStreamed = async (files: File[]): Promise<Uint8Array> => {
  const mergedDoc = await PDFDocument.create();

  // Process one file at a time — don't hold all parsed docs simultaneously
  for (const file of files) {
    const bytes = new Uint8Array(await file.arrayBuffer());
    const srcDoc = await PDFDocument.load(bytes);
    const indices = srcDoc.getPageIndices();
    const pages = await mergedDoc.copyPagesFrom(srcDoc, indices);
    pages.forEach(p => mergedDoc.addPage(p));
    // srcDoc goes out of scope here — eligible for GC
  }

  return mergedDoc.save();
};
```

Processing files sequentially (not `Promise.all`) means only one source document is fully parsed at a time. The previous one becomes garbage-collectable before the next is loaded.

## Handling Encrypted PDFs

pdf-lib throws on encrypted PDFs:

```typescript
try {
  const srcDoc = await PDFDocument.load(bytes);
} catch (e) {
  const msg = String(e);
  if (msg.includes('encrypted') || msg.includes('password')) {
    throw new Error(`"${file.name}" is password-protected. Remove the password first.`);
  }
  throw e;
}
```

Surfacing a per-file error (with the filename) is better than a generic failure — the user knows which file to fix.

## Output File Naming

```typescript
const triggerDownload = (bytes: Uint8Array) => {
  const blob = new Blob([bytes], { type: 'application/pdf' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'merged.pdf';
  a.click();
  URL.revokeObjectURL(url);
};
```

## Key Gotchas

| Problem | Solution |
|---|---|
| Can't add pages across documents directly | Always use `copyPagesFrom` |
| Loading all PDFs at once spikes memory | Process sequentially, let GC reclaim each |
| Drag reorder breaks React reconciler | Use stable `id` (not index) as key |
| Encrypted PDFs throw generic error | Catch, check message, report per-file |
| Empty merge result | Guard: don't call if `entries.length === 0` |

The full tool is live at the [Merge PDF](https://ultimatetools.io/tools/pdf-tools/merge-pdf/) tool — drop files, drag to order, download the combined result.