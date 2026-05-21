# How to Optimize Images for Web Performance — Format, Size, and Compression Guide

Images are typically the largest assets on a web page and the biggest contributor to slow load times. Google's Core Web Vitals — specifically LCP (Largest Contentful Paint) — are directly affected by how well your images are optimized.

This guide covers the three things that actually move the needle: choosing the right format, resizing to the right dimensions, and compressing without visible quality loss.

---

## Step 1 — Choose the Right Format

| Format | Best For | Avoid For |
|---|---|---|
| **WebP** | Photos, general web images | IE11 users (rare, but exists) |
| **JPG** | Photos when WebP isn't supported | Logos, icons with transparency |
| **PNG** | Screenshots, logos, images with transparency | Large photos (file size too large) |
| **SVG** | Icons, logos, illustrations | Photos |
| **AVIF** | Best compression, modern browsers only | Broad compatibility needs |

**WebP is the default choice for 2024+.** It delivers 25–35% smaller files than JPG at equivalent visual quality and is supported by all modern browsers.

Convert any image to WebP: [Image Converter — JPG, PNG to WebP, Free](https://ultimatetools.io/tools/image-tools/image-converter/)

---

## Step 2 — Resize to Display Dimensions

The most common mistake: uploading a 4000×3000px photo for a thumbnail that displays at 400×300px. The browser downloads 10× more pixels than it shows.

**Rule:** Export images at 1× or 2× the CSS display size — no larger.

Common display sizes to target:

| Element | CSS Display Width | Export Width (2×) |
|---|---|---|
| Blog hero image | 1200px | 1200px (single density fine) |
| Thumbnail | 400px | 800px |
| Open Graph image | 1200×630px | 1200×630px |
| Avatar / profile | 80px | 160px |

Resize to exact dimensions: [Image Resizer — Pixels, Free, No Upload](https://ultimatetools.io/tools/image-tools/image-resizer/)

---

## Step 3 — Compress Without Visible Quality Loss

After choosing the right format and resizing, compression removes further redundant data from the file.

**Target file sizes for web:**
- Hero image / banner: under 200KB
- Blog inline image: under 100KB
- Thumbnail: under 30KB
- Icon (PNG): under 10KB

Compress images in browser: [Image Compressor — Free, No Upload](https://ultimatetools.io/tools/image-tools/image-compressor/)

The tool uses browser-side compression — your files never leave your device.

---

## Step 4 — Use `loading="lazy"` for Below-the-Fold Images

This is a one-line HTML change, not an image tool — but it's worth including:

```html
<img src="image.webp" loading="lazy" width="800" height="600" alt="description">
```

The browser skips loading images below the fold until the user scrolls. This reduces initial page weight for image-heavy pages.

Always include `width` and `height` attributes — they prevent layout shift (CLS) while the image loads.

---

## Step 5 — Serve Responsive Images With `srcset`

For images displayed at different sizes on different screen widths:

```html
<img
  srcset="image-400.webp 400w, image-800.webp 800w, image-1200.webp 1200w"
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  src="image-800.webp"
  alt="description"
/>
```

The browser picks the right size for the device. Mobile users download the 400px image; desktop users get 1200px. Export all three sizes from the [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/).

---

## Quick Audit Checklist

Before deploying any page with images:

- [ ] Format: WebP or JPG (not PNG for photos)
- [ ] Dimensions match display size (check in DevTools → Elements → hover the image)
- [ ] File size under limits above (check in Network tab)
- [ ] `loading="lazy"` on below-fold images
- [ ] `width` and `height` attributes set on all `<img>` tags
- [ ] `alt` text on all images

---

## Related Tools

- [Image Compressor](https://ultimatetools.io/tools/image-tools/image-compressor/) — reduce file size without visible quality loss
- [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/) — resize to exact pixel dimensions
- [Image Converter](https://ultimatetools.io/tools/image-tools/image-converter/) — convert JPG/PNG to WebP

---

Run these three steps — right format, right size, compressed — and most pages will see a meaningful LCP improvement without touching a line of application code.