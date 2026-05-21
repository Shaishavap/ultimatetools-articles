# html2canvas vs dom-to-image vs html-to-image — Browser Screenshot Library Comparison

Taking a screenshot of a DOM element in the browser sounds simple. In practice, it involves re-implementing a significant subset of CSS rendering in JavaScript — and the three main libraries that do this have meaningfully different trade-offs.

This post compares html2canvas, dom-to-image (and its fork dom-to-image-more), and html-to-image — based on feature support, accuracy, performance, and bundle size.

---

## The Core Challenge

Browsers don't expose a native "capture this element as an image" API for arbitrary DOM content. The `html2canvas` approach involves:

1. Walking the DOM tree
2. Reading computed styles for each element
3. Re-drawing each element onto an HTML5 Canvas
4. Exporting the canvas as a PNG or JPEG

The alternative approach (used by dom-to-image and html-to-image) uses a different trick:

1. Clone the DOM subtree
2. Inline all styles and resources
3. Serialize the clone to an SVG using a `<foreignObject>` element
4. Render the SVG to a Canvas using an Image element

Each approach has different failure modes.

---

## Library Overview

### html2canvas

- **GitHub**: timmywil/html2canvas
- **Stars**: ~30k
- **Bundle size**: ~200KB minified
- **Approach**: Canvas re-implementation of CSS rendering
- **Actively maintained**: Yes (2024 releases)

The original. Widely used, widely tested. The Canvas approach means it has the most predictable behaviour across browsers, but it also means CSS feature coverage depends on what html2canvas has implemented.

**CSS coverage limitations:**
- CSS Grid: partial support
- CSS Custom Properties (`var()`): supported in modern versions
- CSS filters: partial support
- `position: fixed` elements: known issues
- Web fonts: requires explicit configuration

### dom-to-image / dom-to-image-more

- **GitHub**: 1904labs/dom-to-image-more (active fork)
- **Stars**: 10k+ (combined)
- **Bundle size**: ~50KB minified
- **Approach**: SVG foreignObject
- **Actively maintained**: dom-to-image-more is; original dom-to-image is largely abandoned

The SVG foreignObject approach means the browser's own rendering engine handles CSS — so CSS Grid, flexbox, and custom properties just work (as long as the browser supports them). The failure mode is different: `<foreignObject>` rendering has cross-browser inconsistencies, particularly in Firefox and Safari.

**Known issues:**
- Images from external domains may not render (CORS)
- Web fonts require explicit preloading
- Firefox `foreignObject` rendering can differ from Chrome

### html-to-image

- **GitHub**: bubkoo/html-to-image
- **Stars**: 5k+
- **Bundle size**: ~25KB minified
- **Approach**: SVG foreignObject (similar to dom-to-image)
- **TypeScript**: Yes, fully typed
- **Actively maintained**: Yes

Essentially a TypeScript rewrite and modernisation of dom-to-image. Cleaner API, better TypeScript support, smaller bundle. Same underlying approach, similar trade-offs.

```typescript
import { toPng, toJpeg, toBlob, toPixelData, toSvg } from 'html-to-image';

const dataUrl = await toPng(document.getElementById('my-node'));
```

vs html2canvas:

```javascript
const canvas = await html2canvas(document.getElementById('my-node'));
const dataUrl = canvas.toDataURL('image/png');
```

---

## Side-by-Side Comparison

| Feature | html2canvas | dom-to-image-more | html-to-image |
|---|---|---|---|
| Approach | Canvas re-draw | SVG foreignObject | SVG foreignObject |
| Bundle size | ~200KB | ~50KB | ~25KB |
| TypeScript | Partial | No | Yes |
| CSS Grid | Partial | Full (browser-rendered) | Full (browser-rendered) |
| CSS filters | Partial | Browser-rendered | Browser-rendered |
| Cross-browser consistency | High | Medium | Medium |
| Firefox support | Good | Issues with foreignObject | Issues with foreignObject |
| Safari support | Good | Variable | Variable |
| External images (CORS) | CORS proxy needed | CORS proxy needed | CORS proxy needed |
| Web fonts | Configurable | Requires preloading | Requires preloading |
| Active maintenance | Yes | Yes | Yes |
| Output formats | PNG, JPEG | PNG, JPEG, SVG | PNG, JPEG, SVG, Blob |

---

## Which One Should You Use?

**Use html2canvas if:**
- You need consistent results across all browsers including Firefox and Safari
- Your UI uses straightforward CSS (without heavy use of Grid or custom properties)
- You've already been using it and it's working

**Use html-to-image if:**
- You're in a TypeScript project and want full typing
- Your UI uses modern CSS (Grid, custom properties, filters) and you're targeting Chrome/Edge primarily
- Bundle size matters
- You need SVG output as well as PNG/JPEG

**Use dom-to-image-more if:**
- You're already using dom-to-image and want a maintained fork
- You don't need TypeScript

---

## Common Failure Modes and Fixes

### External images not rendering

All three libraries have CORS issues with external images. Solution: use a CORS proxy server, or convert images to data URLs before rendering.

```javascript
// Pre-convert images before capture
async function imageToDataUrl(img) {
  return new Promise((resolve) => {
    const canvas = document.createElement('canvas');
    canvas.width = img.naturalWidth;
    canvas.height = img.naturalHeight;
    canvas.getContext('2d').drawImage(img, 0, 0);
    resolve(canvas.toDataURL());
  });
}
```

### Web fonts not loading

Load fonts explicitly before capture:

```javascript
await document.fonts.ready; // Wait for all fonts to load
const dataUrl = await toPng(element);
```

### Blurry output on high-DPI displays

Apply a scale multiplier:

```javascript
// html-to-image
const dataUrl = await toPng(element, {
  pixelRatio: window.devicePixelRatio
});

// html2canvas
const canvas = await html2canvas(element, {
  scale: window.devicePixelRatio
});
```

---

## Zero-Dependency Alternative

If your use case is simple — capturing a static page or a known DOM structure — and you want zero JavaScript overhead, try the browser-based [HTML to Image Converter](https://ultimatetools.io/tools/image-tools/html-to-image/) at Ultimate Tools. It uses dom-to-image under the hood, runs entirely in the browser, and handles the common failure modes (font loading, scale, CORS-free local content) automatically.

For programmatic use in your own app: html-to-image is the current best choice for TypeScript projects targeting modern browsers. html2canvas remains the most reliable cross-browser option.

---

**Useful links:**
- [html2canvas documentation](https://html2canvas.hertzen.com/)
- [html-to-image on npm](https://www.npmjs.com/package/html-to-image)
- [dom-to-image-more on npm](https://www.npmjs.com/package/dom-to-image-more)
- [HTML to Image Converter](https://ultimatetools.io/tools/image-tools/html-to-image/) — browser tool, no code needed
- [Image Compressor](https://ultimatetools.io/tools/image-tools/image-compressor/) — compress your exported PNG/JPEG before using it