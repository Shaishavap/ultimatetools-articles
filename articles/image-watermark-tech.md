# Building a Browser-Based Image Watermark Tool with Canvas API — Text Overlay, Opacity, Auto-Shadow

Adding a text watermark to an image is 20 lines of Canvas API. Making it look good — opacity, adaptive shadow, position grid, live preview — takes a bit more thought. Here's how the [Watermark Image](https://ultimatetools.io/tools/image-tools/watermark-image/) tool is built.

## Core: Drawing Text on Canvas

The fundamental operation:

```typescript
function drawWatermark(
  canvas: HTMLCanvasElement,
  img: HTMLImageElement,
  options: WatermarkOptions
) {
  const ctx = canvas.getContext('2d')!

  // Draw base image
  ctx.drawImage(img, 0, 0)

  // Configure text style
  ctx.font = `bold ${options.fontSize}px Arial, sans-serif`
  ctx.globalAlpha = options.opacity  // 0.0–1.0
  ctx.fillStyle = options.color

  // Position and draw
  const { x, y } = getPosition(canvas, ctx, options)
  ctx.fillText(options.text, x, y)

  // Reset alpha
  ctx.globalAlpha = 1.0
}
```

`globalAlpha` applies to everything drawn after it's set — set it before `fillText`, reset it after. Forgetting to reset causes all subsequent canvas operations to inherit the opacity.

## Position Grid: 9 Positions

A 3×3 grid gives 9 positions (top-left through bottom-right). Map the selected position to canvas coordinates:

```typescript
type Position =
  | 'top-left'    | 'top-center'    | 'top-right'
  | 'middle-left' | 'middle-center' | 'middle-right'
  | 'bottom-left' | 'bottom-center' | 'bottom-right'

const PADDING = 20 // px from edge

function getPosition(
  canvas: HTMLCanvasElement,
  ctx: CanvasRenderingContext2D,
  options: WatermarkOptions
): { x: number; y: number } {
  const metrics = ctx.measureText(options.text)
  const textWidth = metrics.width
  const textHeight = options.fontSize

  const positions: Record<Position, { x: number; y: number }> = {
    'top-left':      { x: PADDING, y: PADDING + textHeight },
    'top-center':    { x: (canvas.width - textWidth) / 2, y: PADDING + textHeight },
    'top-right':     { x: canvas.width - textWidth - PADDING, y: PADDING + textHeight },
    'middle-left':   { x: PADDING, y: (canvas.height + textHeight) / 2 },
    'middle-center': { x: (canvas.width - textWidth) / 2, y: (canvas.height + textHeight) / 2 },
    'middle-right':  { x: canvas.width - textWidth - PADDING, y: (canvas.height + textHeight) / 2 },
    'bottom-left':   { x: PADDING, y: canvas.height - PADDING },
    'bottom-center': { x: (canvas.width - textWidth) / 2, y: canvas.height - PADDING },
    'bottom-right':  { x: canvas.width - textWidth - PADDING, y: canvas.height - PADDING },
  }

  return positions[options.position]
}
```

`ctx.measureText(text).width` gives the rendered text width in pixels for the current font settings — call this *after* setting `ctx.font`. For y-position, canvas text baseline is at the bottom of the em box, so add `fontSize` to vertical positions that start from the top.

## Adaptive Shadow for Readability

A white watermark on a bright sky disappears. A black watermark on a dark photo disappears. The solution: always add a contrasting shadow automatically.

```typescript
function applyAdaptiveShadow(ctx: CanvasRenderingContext2D, textColor: string) {
  const isLight = isLightColor(textColor)

  ctx.shadowColor = isLight ? 'rgba(0,0,0,0.6)' : 'rgba(255,255,255,0.6)'
  ctx.shadowBlur = 4
  ctx.shadowOffsetX = 1
  ctx.shadowOffsetY = 1
}

function isLightColor(hex: string): boolean {
  // Parse hex to RGB
  const r = parseInt(hex.slice(1, 3), 16)
  const g = parseInt(hex.slice(3, 5), 16)
  const b = parseInt(hex.slice(5, 7), 16)
  // Perceived luminance formula
  const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255
  return luminance > 0.5
}
```

Light text color → dark shadow. Dark text color → light shadow. The shadow is always present, always contrasting, automatically.

Reset shadow after drawing to avoid it bleeding into subsequent canvas operations:

```typescript
function resetShadow(ctx: CanvasRenderingContext2D) {
  ctx.shadowColor = 'transparent'
  ctx.shadowBlur = 0
  ctx.shadowOffsetX = 0
  ctx.shadowOffsetY = 0
}
```

## Full Watermark Draw Function

```typescript
interface WatermarkOptions {
  text: string
  fontSize: number        // px
  color: string           // hex
  opacity: number         // 0.0–1.0
  position: Position
}

function renderWatermark(
  img: HTMLImageElement,
  options: WatermarkOptions
): HTMLCanvasElement {
  const canvas = document.createElement('canvas')
  canvas.width = img.naturalWidth
  canvas.height = img.naturalHeight

  const ctx = canvas.getContext('2d')!

  // 1. Draw base image
  ctx.drawImage(img, 0, 0)

  // 2. Configure text rendering
  ctx.font = `bold ${options.fontSize}px Arial, sans-serif`
  ctx.fillStyle = options.color
  ctx.globalAlpha = options.opacity

  // 3. Apply adaptive shadow
  applyAdaptiveShadow(ctx, options.color)

  // 4. Measure and position
  const { x, y } = getPosition(canvas, ctx, options)

  // 5. Draw text
  ctx.fillText(options.text, x, y)

  // 6. Cleanup
  ctx.globalAlpha = 1.0
  resetShadow(ctx)

  return canvas
}
```

## Live Preview: Debounced Re-render

For the live preview, re-render on every options change. Debounce to avoid thrashing on slider drag:

```typescript
import { useEffect, useRef, useState } from 'react'

function useWatermarkPreview(img: HTMLImageElement | null, options: WatermarkOptions) {
  const [previewUrl, setPreviewUrl] = useState<string | null>(null)
  const debounceRef = useRef<ReturnType<typeof setTimeout>>()

  useEffect(() => {
    if (!img) return

    clearTimeout(debounceRef.current)
    debounceRef.current = setTimeout(() => {
      const canvas = renderWatermark(img, options)
      const url = canvas.toDataURL('image/jpeg', 0.85)

      setPreviewUrl(prev => {
        if (prev) URL.revokeObjectURL(prev) // cleanup old blob URLs
        return url
      })
    }, 50) // 50ms debounce

    return () => clearTimeout(debounceRef.current)
  }, [img, options])

  return previewUrl
}
```

50ms debounce is fast enough to feel responsive while avoiding a render on every pixel of slider movement.

Note: using `toDataURL` for preview (synchronous, simpler) and `toBlob` for download (async, smaller memory footprint for large images).

## Font Size Relative to Image

A fixed 40px font size looks tiny on a 4000px image and enormous on a 400px image. Scale to a percentage of the image's short side:

```typescript
function getDefaultFontSize(img: HTMLImageElement): number {
  const shortSide = Math.min(img.naturalWidth, img.naturalHeight)
  return Math.round(shortSide * 0.04) // 4% of short side
  // 400px image  → 16px
  // 1920px image → 76px
  // 4000px image → 160px
}
```

Expose a slider (e.g. 10–200px) with this as the default. The user can adjust from the baseline.

## Download

```typescript
async function downloadWatermarked(
  img: HTMLImageElement,
  options: WatermarkOptions,
  originalFile: File
) {
  const canvas = renderWatermark(img, options)

  const mimeType = originalFile.type === 'image/png' ? 'image/png' : 'image/jpeg'
  const quality = mimeType === 'image/jpeg' ? 0.92 : undefined

  const blob = await new Promise<Blob>((resolve, reject) =>
    canvas.toBlob(b => b ? resolve(b) : reject(), mimeType, quality)
  )

  const baseName = originalFile.name.replace(/\.[^.]+$/, '')
  const ext = mimeType === 'image/png' ? 'png' : 'jpg'
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `${baseName}-watermarked.${ext}`
  a.click()
  URL.revokeObjectURL(url)
}
```

Preserve PNG format for transparency. Default to JPEG for everything else to keep file sizes reasonable.

## What About Image Watermarks (Logo Overlay)?

Text watermarks are covered by `fillText`. Logo watermarks require drawing a second image on top of the first:

```typescript
function drawLogoWatermark(
  ctx: CanvasRenderingContext2D,
  logo: HTMLImageElement,
  x: number,
  y: number,
  maxWidth: number,
  opacity: number
) {
  // Scale logo to maxWidth while preserving aspect ratio
  const scale = Math.min(1, maxWidth / logo.naturalWidth)
  const w = logo.naturalWidth * scale
  const h = logo.naturalHeight * scale

  ctx.globalAlpha = opacity
  ctx.drawImage(logo, x, y, w, h)
  ctx.globalAlpha = 1.0
}
```

Same pattern — `globalAlpha` for opacity, `drawImage` for rendering. The current tool focuses on text watermarks; logo support could be added with the same canvas approach.

---

Try it: [Watermark Image → ultimatetools.io](https://ultimatetools.io/tools/image-tools/watermark-image/)

*Part of [Ultimate Tools](https://ultimatetools.io/) — free, privacy-first browser tools.*