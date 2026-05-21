# Building a JPG to PDF Converter in the Browser with pdf-lib — Image Embedding, Page Sizing, and Drag-to-Reorder

Converting images to PDF in the browser requires three things: reading files without a backend, embedding them correctly into PDF pages, and giving the user a way to control page order before converting.

Here's how the [JPG to PDF tool](https://ultimatetools.io/tools/pdf-tools/jpg-to-pdf/) at Ultimate Tools handles each of these.

## File Reading: FileReader vs. arrayBuffer()

Each image needs to be read into memory before embedding. The tool uses the modern `arrayBuffer()` approach rather than the older `FileReader` API:

```typescript
const arrayBuffer = await file.arrayBuffer();
const bytes = new Uint8Array(arrayBuffer);
```

This is cleaner than `FileReader` (no callback wrapping needed) and supported in all modern browsers. The bytes are passed directly to pdf-lib's embed functions.

## Detecting Image Format

pdf-lib has separate methods for PNG and JPEG. The format must be detected before embedding:

```typescript
const embedImage = async (pdfDoc: PDFDocument, file: File) => {
  const bytes = new Uint8Array(await file.arrayBuffer());

  const mimeType = file.type.toLowerCase();

  if (mimeType === 'image/png') {
    return pdfDoc.embedPng(bytes);
  } else if (mimeType === 'image/jpeg' || mimeType === 'image/jpg') {
    return pdfDoc.embedJpg(bytes);
  } else {
    // WebP and other formats: draw to canvas first, export as PNG
    return embedViaCanvas(pdfDoc, file);
  }
};
```

pdf-lib natively supports JPEG and PNG. For WebP, GIF, and other formats, the image must be drawn to a canvas first and exported as PNG:

```typescript
const embedViaCanvas = async (pdfDoc: PDFDocument, file: File): Promise<PDFImage> => {
  const bitmap = await createImageBitmap(file);
  const canvas = document.createElement('canvas');
  canvas.width = bitmap.width;
  canvas.height = bitmap.height;
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(bitmap, 0, 0);

  const pngDataUrl = canvas.toDataURL('image/png');
  const base64 = pngDataUrl.split(',')[1];
  const pngBytes = Uint8Array.from(atob(base64), c => c.charCodeAt(0));
  return pdfDoc.embedPng(pngBytes);
};
```

`createImageBitmap()` handles decoding of WebP and GIF natively — the canvas just re-encodes it as PNG for pdf-lib.

## Page Sizing: Match the Image's Aspect Ratio

Each image becomes one PDF page. The page dimensions should match the image so there's no letterboxing or distortion.

pdf-lib works in PDF points (72 points = 1 inch). The image's pixel dimensions are used directly as the page size — the PDF is rendered at 72 DPI by default, which means 1 pixel = 1 point:

```typescript
const addImagePage = (pdfDoc: PDFDocument, image: PDFImage) => {
  const { width, height } = image.scale(1);

  const page = pdfDoc.addPage([width, height]);

  page.drawImage(image, {
    x: 0,
    y: 0,
    width,
    height,
  });
};
```

`image.scale(1)` returns the image's natural dimensions in PDF points. Using these as the page size guarantees the image fills the entire page without any margins or white space.

## Full Conversion Flow

```typescript
const convertToPdf = async (files: File[]): Promise<Uint8Array> => {
  const pdfDoc = await PDFDocument.create();

  for (const file of files) {
    const image = await embedImage(pdfDoc, file);
    addImagePage(pdfDoc, image);
  }

  return pdfDoc.save();
};
```

One `PDFDocument`, one page per image, in the order the user arranged them.

## Drag-to-Reorder with React State

The UI maintains an ordered list of file entries. Dragging reorders the array:

```typescript
type ImageEntry = {
  id: string;
  file: File;
  previewUrl: string;
};

const [images, setImages] = useState<ImageEntry[]>([]);

const moveImage = (fromIndex: number, toIndex: number) => {
  setImages(prev => {
    const next = [...prev];
    const [moved] = next.splice(fromIndex, 1);
    next.splice(toIndex, 0, moved);
    return next;
  });
};
```

For drag handling, pointer events are used:

```typescript
const onDragStart = (e: DragEvent<HTMLDivElement>, index: number) => {
  dragIndexRef.current = index;
};

const onDrop = (e: DragEvent<HTMLDivElement>, index: number) => {
  e.preventDefault();
  if (dragIndexRef.current !== null && dragIndexRef.current !== index) {
    moveImage(dragIndexRef.current, index);
  }
  dragIndexRef.current = null;
};
```

## Triggering the Download

pdf-lib's `save()` returns a `Uint8Array`. To download it:

```typescript
const triggerDownload = (bytes: Uint8Array, filename: string) => {
  const blob = new Blob([bytes], { type: 'application/pdf' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
};
```

`URL.revokeObjectURL()` is important — it releases the object URL after the download triggers, preventing a memory leak.

## Preview Thumbnails

Each image gets a local preview URL for the drag-to-reorder list:

```typescript
const addFiles = (newFiles: File[]) => {
  const entries: ImageEntry[] = newFiles.map(file => ({
    id: crypto.randomUUID(),
    file,
    previewUrl: URL.createObjectURL(file),
  }));
  setImages(prev => [...prev, ...entries]);
};

// Cleanup on component unmount
useEffect(() => {
  return () => {
    images.forEach(img => URL.revokeObjectURL(img.previewUrl));
  };
}, []);
```

## Key Gotchas

| Problem | Solution |
|---|---|
| WebP not supported by pdf-lib | Draw to canvas, export as PNG |
| Page size doesn't match image | Use `image.scale(1)` for natural dimensions |
| Memory leak from object URLs | `revokeObjectURL` on unmount and after download |
| GIF only converts first frame | `createImageBitmap` decodes the first frame by default |
| Large images cause slow conversion | Process sequentially (one at a time) to avoid memory spikes |

The full tool is live at the [JPG to PDF converter](https://ultimatetools.io/tools/pdf-tools/jpg-to-pdf/) — drop your images, drag to reorder, download your PDF.