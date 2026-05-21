# Building a Browser Crop Image Tool — Display-to-Source Coordinate Scaling and Aspect Ratio Presets

The [Crop Image](https://ultimatetools.io/tools/image-tools/crop-image/) tool at Ultimate Tools crops images to exact selections in the browser using the Canvas API. The non-obvious problem: mouse drag events give coordinates in the display space (the scaled-down preview), but the crop must be applied to the full-resolution source image.

---

## The Two-Coordinate-Space Problem

The user sees the image at preview scale — maybe 600px wide on screen. The original image is 4000px wide. A drag from `x=100` to `x=400` in display space covers 300px of the preview, but 2000px of the source image.

```typescript
function getScaleFactors(
  img: HTMLImageElement,
  displayWidth: number,
  displayHeight: number
) {
  return {
    scaleX: img.naturalWidth / displayWidth,
    scaleY: img.naturalHeight / displayHeight,
  };
}
```

Every drag coordinate gets multiplied by these scale factors before applying to canvas:

```typescript
function toSourceCoords(
  displayRect: { x: number; y: number; width: number; height: number },
  scale: { scaleX: number; scaleY: number }
) {
  return {
    x: Math.round(displayRect.x * scale.scaleX),
    y: Math.round(displayRect.y * scale.scaleY),
    width:  Math.round(displayRect.width  * scale.scaleX),
    height: Math.round(displayRect.height * scale.scaleY),
  };
}
```

Skip this step and every crop is off by the scale factor — visually close, but subtly wrong in a way users notice immediately ("it's not cutting where I dragged it").

---

## Drag Handler

```typescript
interface Rect {
  x: number; y: number;
  width: number; height: number;
}

function useCropDrag(containerRef: React.RefObject<HTMLDivElement>) {
  const [selection, setSelection] = useState<Rect | null>(null);
  const dragging = useRef(false);
  const startPoint = useRef({ x: 0, y: 0 });

  function getRelativePoint(e: MouseEvent | TouchEvent): { x: number; y: number } {
    const rect = containerRef.current!.getBoundingClientRect();
    const clientX = 'touches' in e ? e.touches[0].clientX : e.clientX;
    const clientY = 'touches' in e ? e.touches[0].clientY : e.clientY;
    return {
      x: Math.max(0, Math.min(clientX - rect.left, rect.width)),
      y: Math.max(0, Math.min(clientY - rect.top, rect.height)),
    };
  }

  function onMouseDown(e: React.MouseEvent) {
    dragging.current = true;
    const pt = getRelativePoint(e.nativeEvent);
    startPoint.current = pt;
    setSelection({ x: pt.x, y: pt.y, width: 0, height: 0 });
  }

  function onMouseMove(e: React.MouseEvent) {
    if (!dragging.current) return;
    const pt = getRelativePoint(e.nativeEvent);
    const x = Math.min(pt.x, startPoint.current.x);
    const y = Math.min(pt.y, startPoint.current.y);
    setSelection({
      x, y,
      width:  Math.abs(pt.x - startPoint.current.x),
      height: Math.abs(pt.y - startPoint.current.y),
    });
  }

  function onMouseUp() { dragging.current = false; }

  return { selection, onMouseDown, onMouseMove, onMouseUp };
}
```

`Math.min` / `Math.abs` lets the user drag in any direction — not just top-left to bottom-right. Without this, dragging up or left produces a negative-dimension rect and the selection breaks.

The point clamping (`Math.max(0, Math.min(..., rect.width))`) keeps the selection inside the image even when the user drags the mouse outside the container.

---

## Aspect Ratio Lock

When a ratio is selected, constrain the selection to that ratio on every mouse-move:

```typescript
const RATIOS: Record<string, number | null> = {
  '16:9': 16 / 9,
  '4:3':  4 / 3,
  '1:1':  1,
  '9:16': 9 / 16,
  'free': null,
};

function applyRatio(
  width: number,
  height: number,
  ratio: number | null
): { width: number; height: number } {
  if (!ratio) return { width, height };

  if (width / height > ratio) {
    return { width: height * ratio, height };
  }
  return { width, height: width / ratio };
}
```

Call this inside `onMouseMove` before updating state:

```typescript
const constrained = applyRatio(rawWidth, rawHeight, selectedRatio);
setSelection({ x, y, ...constrained });
```

---

## Applying the Crop

```typescript
function cropToCanvas(
  img: HTMLImageElement,
  sourceRect: Rect
): HTMLCanvasElement {
  const canvas = document.createElement('canvas');
  canvas.width  = sourceRect.width;
  canvas.height = sourceRect.height;

  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(
    img,
    sourceRect.x, sourceRect.y, sourceRect.width, sourceRect.height, // source
    0, 0, sourceRect.width, sourceRect.height                         // dest
  );

  return canvas;
}
```

The nine-argument `drawImage` does the crop in one call: read a rectangle from the source image, draw it at `(0,0)` filling the entire canvas. The output canvas is exactly the cropped dimensions — no scaling.

---

## Export

```typescript
function exportCrop(canvas: HTMLCanvasElement, filename: string): void {
  canvas.toBlob((blob) => {
    if (!blob) return;
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename.replace(/\.[^.]+$/, '-cropped.png');
    a.click();
    URL.revokeObjectURL(url);
  }, 'image/png');
}
```

Export as PNG — lossless output regardless of the input format. A cropped JPEG re-exported as JPEG would apply a second lossy compression pass.

---

## Full Flow

```typescript
async function handleCrop(file: File, displaySelection: Rect, displaySize: { w: number; h: number }) {
  const img = await loadImageFromFile(file);
  const scale = getScaleFactors(img, displaySize.w, displaySize.h);
  const sourceRect = toSourceCoords(displaySelection, scale);
  const canvas = cropToCanvas(img, sourceRect);
  exportCrop(canvas, file.name);
}
```

The full [Crop Image](https://ultimatetools.io/tools/image-tools/crop-image/) tool — free drag selection, aspect ratio presets, full-resolution PNG export — is live at Ultimate Tools. No account, no server upload.