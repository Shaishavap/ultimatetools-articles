# Three Canvas Blur Effects Without a Library — Gaussian Convolution, Pixelate, and CSS Backdrop Blur

Blur is one of the trickier image effects to implement in the browser because the Canvas API doesn't expose a native Gaussian blur for per-pixel manipulation — only CSS `filter: blur()` and `ctx.filter`. Doing it properly on a canvas requires convolution.

Here's how the [Image Blur](https://ultimatetools.io/tools/image-tools/blur-image/) tool at Ultimate Tools implements three distinct blur effects: Gaussian convolution, pixelate, and backdrop blur.

## Effect 1: Gaussian Blur via Convolution Kernel

Gaussian blur works by replacing each pixel's value with a weighted average of its neighbors. The weights follow a Gaussian (bell curve) distribution — pixels closer to the center contribute more.

### Generating the Kernel

```typescript
const generateGaussianKernel = (radius: number): number[][] => {
  const size = radius * 2 + 1;
  const sigma = radius / 3;
  const kernel: number[][] = [];
  let sum = 0;

  for (let y = -radius; y <= radius; y++) {
    const row: number[] = [];
    for (let x = -radius; x <= radius; x++) {
      const value = Math.exp(-(x * x + y * y) / (2 * sigma * sigma));
      row.push(value);
      sum += value;
    }
    kernel.push(row);
  }

  // Normalize so all weights sum to 1
  return kernel.map(row => row.map(v => v / sum));
};
```

### Applying the Kernel

```typescript
const applyGaussianBlur = (imageData: ImageData, radius: number): ImageData => {
  const kernel = generateGaussianKernel(radius);
  const { data, width, height } = imageData;
  const output = new Uint8ClampedArray(data.length);

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      let r = 0, g = 0, b = 0, a = 0;

      for (let ky = -radius; ky <= radius; ky++) {
        for (let kx = -radius; kx <= radius; kx++) {
          // Clamp to image boundaries (edge extension)
          const px = Math.min(Math.max(x + kx, 0), width - 1);
          const py = Math.min(Math.max(y + ky, 0), height - 1);
          const pi = (py * width + px) * 4;
          const weight = kernel[ky + radius][kx + radius];

          r += data[pi] * weight;
          g += data[pi + 1] * weight;
          b += data[pi + 2] * weight;
          a += data[pi + 3] * weight;
        }
      }

      const oi = (y * width + x) * 4;
      output[oi] = r;
      output[oi + 1] = g;
      output[oi + 2] = b;
      output[oi + 3] = a;
    }
  }

  return new ImageData(output, width, height);
};
```

**Edge handling:** `Math.min(Math.max(..., 0), width - 1)` clamps coordinates to the image boundary. This "edge extension" avoids a black border that would appear if out-of-bounds pixels were treated as zero.

**Performance note:** This is O(width × height × radius²). For large images at high radius, it's slow. For radius > 15, the tool switches to ctx.filter for speed — see the optimization below.

### Performance Optimization: ctx.filter

For large blur radii where convolution becomes expensive:

```typescript
const blurViaCtxFilter = (
  srcCanvas: HTMLCanvasElement,
  radius: number
): HTMLCanvasElement => {
  const dst = document.createElement('canvas');
  dst.width = srcCanvas.width;
  dst.height = srcCanvas.height;
  const ctx = dst.getContext('2d')!;

  ctx.filter = `blur(${radius}px)`;
  ctx.drawImage(srcCanvas, 0, 0);
  ctx.filter = 'none';

  return dst;
};
```

`ctx.filter` applies a CSS-equivalent Gaussian blur during `drawImage`. It's GPU-accelerated in modern browsers and handles large radii in milliseconds. The tradeoff: it doesn't expose per-pixel control (can't do selective blur), but for full-image blur it's ideal.

## Effect 2: Pixelate

Pixelate divides the image into rectangular blocks and fills each block with the average color of pixels in that area:

```typescript
const applyPixelate = (imageData: ImageData, blockSize: number): ImageData => {
  const { data, width, height } = imageData;
  const output = new Uint8ClampedArray(data.length);

  for (let y = 0; y < height; y += blockSize) {
    for (let x = 0; x < width; x += blockSize) {
      // Average colors in this block
      let r = 0, g = 0, b = 0, a = 0, count = 0;

      for (let dy = 0; dy < blockSize && y + dy < height; dy++) {
        for (let dx = 0; dx < blockSize && x + dx < width; dx++) {
          const pi = ((y + dy) * width + (x + dx)) * 4;
          r += data[pi];
          g += data[pi + 1];
          b += data[pi + 2];
          a += data[pi + 3];
          count++;
        }
      }

      r = Math.round(r / count);
      g = Math.round(g / count);
      b = Math.round(b / count);
      a = Math.round(a / count);

      // Fill the block with the average color
      for (let dy = 0; dy < blockSize && y + dy < height; dy++) {
        for (let dx = 0; dx < blockSize && x + dx < width; dx++) {
          const oi = ((y + dy) * width + (x + dx)) * 4;
          output[oi] = r;
          output[oi + 1] = g;
          output[oi + 2] = b;
          output[oi + 3] = a;
        }
      }
    }
  }

  return new ImageData(output, width, height);
};
```

This is O(width × height) regardless of block size — each pixel is read once and written once.

## Effect 3: Backdrop (Heavy) Blur

Backdrop blur is a heavy Gaussian blur at a large radius (40–80px) — the image resolves into abstract color fields with no recognizable detail. The implementation uses `ctx.filter` at high blur values:

```typescript
const applyBackdropBlur = (srcCanvas: HTMLCanvasElement, intensity: number): HTMLCanvasElement => {
  const radius = 20 + intensity * 3; // intensity 0-20 → radius 20-80px
  return blurViaCtxFilter(srcCanvas, radius);
};
```

At radius 60+, faces, text, and shapes are completely unrecognizable. This makes it useful for background generation and for creating "frosted glass" effects.

## Reading and Writing Image Data

All three effects use the same pipeline:

```typescript
const applyEffect = async (
  file: File,
  effect: 'gaussian' | 'pixelate' | 'backdrop',
  intensity: number
): Promise<string> => {
  const bitmap = await createImageBitmap(file);
  const canvas = document.createElement('canvas');
  canvas.width = bitmap.width;
  canvas.height = bitmap.height;
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(bitmap, 0, 0);

  if (effect === 'gaussian') {
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const blurred = applyGaussianBlur(imageData, intensity);
    ctx.putImageData(blurred, 0, 0);
  } else if (effect === 'pixelate') {
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const pixelated = applyPixelate(imageData, intensity * 2);
    ctx.putImageData(pixelated, 0, 0);
  } else {
    return applyBackdropBlur(canvas, intensity).toDataURL('image/jpeg', 0.92);
  }

  return canvas.toDataURL('image/jpeg', 0.92);
};
```

`createImageBitmap` handles JPEG, PNG, and WebP decoding in a single call.

## Key Comparison

| Effect | Approach | Complexity | Use Case |
|---|---|---|---|
| Gaussian | Convolution kernel | O(n × r²) | Natural softening |
| Pixelate | Block averaging | O(n) | Censoring/anonymizing |
| Backdrop | ctx.filter at high radius | GPU-accelerated | Abstract backgrounds |

The three effects cover the main use cases for browser-based blur — smooth, censor, and abstract. The full tool is live at [Image Blur](https://ultimatetools.io/tools/image-tools/blur-image/).