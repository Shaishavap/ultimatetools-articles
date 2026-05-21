# Building a Browser-Based PDF eSign Tool — Canvas Signature Drawing, Drag-to-Place Overlay, and pdf-lib Embedding

Building a PDF signing tool means solving four distinct problems:

1. **Signature creation** — letting the user draw, type, or upload a signature
2. **Page preview** — rendering the PDF so the user can see where to place the signature
3. **Interactive placement** — drag, resize, rotate the signature over the PDF canvas
4. **Embedding** — converting canvas pixel coordinates back to PDF point coordinates and writing the PNG into the PDF

Here's how the eSign PDF tool at [Ultimate Tools](https://ultimatetools.io/tools/pdf-tools/esign-pdf/) handles each of these.

---

## State Machine

The component uses an explicit state enum to prevent impossible UI states:

```typescript
type AppState = 'idle' | 'previewing' | 'signing' | 'placing' | 'processing' | 'done' | 'error';
```

Flow: `idle` → (file drop) → `previewing` → (open signature pad) → `signing` → (confirm) → `placing` → (apply) → `processing` → `done`

---

## Signature Creation: Three Tabs

### 1. Draw (Canvas-based)

The draw tab uses Pointer Events (not mouse events) for pen, touch, and mouse support:

```typescript
const startDrawing = (e: PointerEvent<HTMLCanvasElement>) => {
  const canvas = sigCanvasRef.current!;
  canvas.setPointerCapture(e.pointerId);  // keeps tracking if pointer leaves canvas
  isDrawingRef.current = true;
  const rect = canvas.getBoundingClientRect();
  const ctx = canvas.getContext('2d')!;
  ctx.beginPath();
  ctx.moveTo(e.clientX - rect.left, e.clientY - rect.top);
};

const draw = (e: PointerEvent<HTMLCanvasElement>) => {
  if (!isDrawingRef.current) return;
  const ctx = canvas.getContext('2d')!;
  ctx.lineTo(e.clientX - rect.left, e.clientY - rect.top);
  ctx.strokeStyle = '#0f172a';
  ctx.lineWidth = 2.5;
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  ctx.stroke();
  setHasStrokes(true);
};
```

`canvas.setPointerCapture(e.pointerId)` is the key call — it ensures pointer move events keep firing even if the pointer leaves the canvas boundary mid-stroke.

### 2. Type (Canvas text rendering)

```typescript
const generateTypeSignature = () => {
  const canvas = document.createElement('canvas');
  canvas.width = 400;
  canvas.height = 150;
  const ctx = canvas.getContext('2d')!;

  ctx.font = '72px "Dancing Script", "Great Vibes", "Brush Script MT", cursive';
  ctx.fillStyle = drawColor;
  ctx.textBaseline = 'middle';
  ctx.fillText(typeText, 20, 75);

  setSigDataUrl(canvas.toDataURL('image/png'));
};
```

The font stack falls back through script fonts — if Dancing Script isn't loaded, it falls to Brush Script MT, then generic cursive.

### 3. Upload

User uploads a PNG/JPG image, which is read into a data URL with `FileReader` and used directly as the signature image.

---

## PDF Page Rendering

The PDF is rendered to a canvas using PDF.js. The scale is calculated to fill the container width:

```typescript
const renderPage = useCallback(async (pdfFile: File, pageNum: number) => {
  const pdfjs = await import('/pdfjs.mjs');
  pdfjs.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs';

  const arrayBuffer = await pdfFile.arrayBuffer();
  const pdf = await pdfjs.getDocument({ data: arrayBuffer }).promise;
  setNumPages(pdf.numPages);

  const page = await pdf.getPage(pageNum);
  const containerWidth = canvas.parentElement?.clientWidth ?? 700;
  const viewport = page.getViewport({ scale: 1 });
  const scale = Math.min((containerWidth - 32) / viewport.width, 1.5);
  const scaledViewport = page.getViewport({ scale });

  canvas.width = scaledViewport.width;
  canvas.height = scaledViewport.height;

  // Store for coordinate transform later
  pdfPageDimsRef.current = { width: viewport.width, height: viewport.height };
  pdfScaleRef.current = scale;

  await page.render({ canvasContext: ctx, viewport: scaledViewport, canvas }).promise;
}, []);
```

Two values are stored in refs — the original PDF page dimensions (in PDF points) and the display scale. These are needed to convert the drag position back to PDF coordinates when embedding.

---

## Drag-to-Place Overlay

The signature appears as a positioned overlay on top of the PDF canvas. It supports drag, four-corner resize, and rotation:

```typescript
const [sigPos, setSigPos] = useState<SigPosition>({
  x: 40, y: 40, width: 220, height: 80, rotation: 0
});

type ActiveTool = 'drag' | 'resize-tl' | 'resize-tr' | 'resize-bl' | 'resize-br' | 'rotate' | null;
const activeToolRef = useRef<ActiveTool>(null);
```

On pointer down, the tool checks which handle was hit:

```typescript
const onOverlayPointerDown = (e: PointerEvent, tool: ActiveTool) => {
  e.stopPropagation();
  activeToolRef.current = tool;
  dragStartRef.current = {
    x: e.clientX, y: e.clientY,
    sigX: sigPos.x, sigY: sigPos.y,
    sigWidth: sigPos.width, sigHeight: sigPos.height,
    sigRotation: sigPos.rotation ?? 0,
  };
  (e.currentTarget as HTMLElement).setPointerCapture(e.pointerId);
};
```

On pointer move, the delta is applied to position or size depending on the active tool.

---

## Coordinate Transform: Canvas Pixels → PDF Points

When the user clicks "Apply", the signature's canvas-pixel position must be converted to PDF point coordinates:

```typescript
const handleApply = async () => {
  // sigPos.x/y are in CSS pixels relative to the rendered canvas
  // pdfScaleRef.current is the CSS pixels / PDF points ratio
  // pdfPageDimsRef.current is the PDF page size in points

  const scale = pdfScaleRef.current;
  const { height: pdfH } = pdfPageDimsRef.current;

  // Convert CSS pixels → PDF points
  const pdfX = sigPos.x / scale;
  const pdfW = sigPos.width / scale;
  const pdfH_sig = sigPos.height / scale;

  // PDF coordinate system: y=0 is bottom-left (opposite of CSS)
  const pdfY = pdfH - (sigPos.y / scale) - pdfH_sig;

  const form = new FormData();
  form.append('file', file);
  form.append('signatureDataUrl', sigDataUrl);
  form.append('x', String(pdfX));
  form.append('y', String(pdfY));
  form.append('width', String(pdfW));
  form.append('height', String(pdfH_sig));
  form.append('rotation', String(sigPos.rotation ?? 0));
  form.append('pageIndex', String(currentPage - 1));
};
```

The y-flip (`pdfH - cssY - sigHeight`) is the critical conversion. CSS has y=0 at the top; PDF has y=0 at the bottom.

---

## Server-Side Embedding (pdf-lib)

```typescript
// app/api/esign-pdf/route.ts
const base64Data = signatureDataUrl.replace(/^data:image\/png;base64,/, '');
const pngBytes = Buffer.from(base64Data, 'base64');

const pdfDoc = await PDFDocument.load(new Uint8Array(arrayBuffer));
const page = pdfDoc.getPages()[safePageIndex];
const sigImage = await pdfDoc.embedPng(pngBytes);
```

Embedding the image with rotation requires an extra step. pdf-lib's `degrees()` is counter-clockwise, while CSS rotation is clockwise. The center of rotation also needs to be corrected:

```typescript
const angleRad = -rotation * (Math.PI / 180);

// Offset of center from bottom-left when rotated
const dx = (clampedW / 2) * Math.cos(angleRad) - (clampedH / 2) * Math.sin(angleRad);
const dy = (clampedW / 2) * Math.sin(angleRad) + (clampedH / 2) * Math.cos(angleRad);

const finalX = cx - dx;
const finalY = cy - dy;

page.drawImage(sigImage, {
  x: finalX,
  y: finalY,
  width: clampedW,
  height: clampedH,
  rotate: degrees(-rotation),  // negate because PDF = CCW, CSS = CW
  opacity: 1,
});
```

Without this correction, the rotated signature would appear offset from where the user placed it.

---

## Key Gotchas

| Problem | Solution |
|---|---|
| Pointer leaves canvas mid-draw | `setPointerCapture(e.pointerId)` |
| PDF y-axis is flipped | `pdfH - cssY - sigH` |
| pdf-lib rotation is CCW, CSS is CW | `degrees(-rotation)` |
| Rotation center offset | Recalculate origin from bounding box center |
| Encrypted PDFs | Catch error, return 400 with user-friendly message |

The full tool is live at [eSign PDF tool](https://ultimatetools.io/tools/pdf-tools/esign-pdf/) — draw your signature, position it on the page, and download the signed PDF. Nothing is stored or uploaded beyond the processing request.